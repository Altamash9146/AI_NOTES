# Section 5: Razorpay + Payments (10 Qs) ⭐ MOST IMPORTANT

> [← Back to index](README.md)


### 1. Complete Razorpay flow — Frontend → Backend → Webhook
```
1. User clicks "Pay"            → Frontend calls POST /api/orders (cart items only, NOT price)
2. Backend                      → calculates amount from DB prices, creates DB order (status PENDING)
                                → razorpay.orders.create({ amount: 49900, currency: 'INR', receipt: dbOrderId, notes: { dbOrderId } })
                                → saves razorpay_order_id in DB, returns { razorpayOrderId, amount, keyId }
3. Frontend                     → opens Razorpay Checkout with order_id
4. User pays                    → Razorpay returns { razorpay_payment_id, razorpay_order_id, razorpay_signature } to handler
5. Frontend                     → POST /api/payments/verify with those 3 values
6. Backend                      → verifies HMAC signature → marks order PAID (idempotent) → returns success
7. Razorpay (server-to-server)  → POST /api/webhooks/razorpay  (payment.captured / order.paid / payment.failed)
8. Backend webhook              → verifies X-Razorpay-Signature → updates DB if not already updated → emits socket event / sends email
```
**Key points to say:** the amount is always computed **on the server**. The frontend success callback is only for UX. **The webhook is the source of truth.** Every update is idempotent.

### 2. Verifying Razorpay signature — how and why?
**Why:** anyone can call your `/verify` API with fake "success" data. The signature proves the response really came from Razorpay and wasn't changed.
```ts
import crypto from 'crypto';

// a) Checkout verification
const body = `${razorpay_order_id}|${razorpay_payment_id}`;
const expected = crypto.createHmac('sha256', process.env.RAZORPAY_KEY_SECRET!).update(body).digest('hex');
const valid = crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(razorpay_signature));

// b) Webhook verification — uses WEBHOOK SECRET + RAW body
app.post('/webhooks/razorpay', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-razorpay-signature'] as string;
  const expected = crypto.createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET!).update(req.body).digest('hex');
  if (expected !== sig) return res.status(400).send('invalid');
  // ...
});
```
- Use the **raw body** for webhooks. Re-stringified JSON can differ, and then the signature won't match. (In a Next.js route handler: `await req.text()`.)
- Use `timingSafeEqual` to avoid timing attacks. Never expose the key secret to the frontend. The SDK also has `validatePaymentVerification` / `validateWebhookSignature`.

### 3. User paid but webhook failed — how to recover?
Several layers of safety:
1. **Two paths update the status:** the frontend `/verify` call AND the webhook. If one fails, the other usually works.
2. **Razorpay retries webhooks** automatically (for about 24 h) if you don't return 2xx. So return 200 quickly and process in a queue.
3. **Reconciliation cron job** (BullMQ, every 5–15 min): find orders that are `PENDING` for more than 10 min → call `razorpay.orders.fetchPayments(orderId)` / `payments.fetch(id)` → if captured, mark them PAID. If authorized but not captured, capture or let auto-refund happen.
4. **Daily settlement reconciliation** against Razorpay reports.
5. Store the raw webhook events in a table so failed ones can be replayed.
6. If the webhook handler errors → retry from the queue → after max attempts go to a DLQ + alert.

### 4. Webhook received twice — handle duplicates
Webhooks are **at-least-once** delivery, so duplicates are normal.
```ts
// table: webhook_events (event_id UNIQUE)
try {
  await prisma.webhookEvent.create({ data: { eventId: req.headers['x-razorpay-event-id'] as string, type, payload } });
} catch (e) {
  if (e.code === 'P2002') return res.status(200).send('duplicate');   // unique violation = already processed
  throw e;
}
// then a state-guarded update
await prisma.order.updateMany({ where: { id, status: 'PENDING' }, data: { status: 'PAID' } });
```
- Dedupe on `x-razorpay-event-id`, and keep `UNIQUE(razorpay_payment_id)` on payments.
- **State machine guard:** only `PENDING → PAID`. If it's already PAID, do nothing. Side effects (email, stock) run only when the update actually changed a row (`count === 1`).
- Always return **200** for duplicates, so Razorpay stops retrying.

### 5. Payment failure / refund flow
**Failure:**
- `payment.failed` webhook / checkout `payment.failed` event → mark the payment attempt FAILED with an error code/reason. The order stays PENDING so the user can **retry with the same Razorpay order** (several payment attempts are allowed per order).
- Release reserved stock after the order expires (e.g. 15 min).

**Refund:**
```ts
const refund = await razorpay.payments.refund(paymentId, { amount: 20000, speed: 'normal', notes: { reason: 'cancelled' } }); // partial or full, in paise
```
- Save a `Refund` row (status `PENDING`) → the `refund.processed` / `refund.failed` webhook updates it → order status `REFUNDED` / `PARTIALLY_REFUNDED`.
- Use an idempotency check so the same order is never refunded twice (check the total refunded so far ≤ amount paid).
- Note: payments that are *authorized but not captured* are auto-refunded by Razorpay after a set period. Turn on **auto-capture** in settings or capture via the API.

### 6. Razorpay Order ID vs Payment ID
| | Order ID (`order_xxx`) | Payment ID (`pay_xxx`) |
|---|---|---|
| Created by | **Your backend**, before checkout | Razorpay, when the user **attempts** a payment |
| Represents | The intent: "collect ₹499 for order #123" | One actual payment attempt |
| Count | One per checkout | **Many per order** (a failed attempt, then a successful retry) |
| Why | Locks the amount (the user can't change it), links to your receipt, prevents duplicate payments | Used for capture, refund, status fetch |

Statuses: Order `created → attempted → paid`. Payment `created → authorized → captured` (or `failed`, `refunded`).

### 7. Race condition: 2 users buying the last item in stock
**Problem:** both read `stock = 1`, both pass the check, both buy, and stock becomes -1.
**Solutions (pick and explain one):**
1. **Atomic conditional update (simplest, best):**
   ```sql
   UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock >= 1;   -- 1 row updated = success, 0 = sold out
   ```
   In Prisma: `updateMany({ where: { id, stock: { gte: qty } }, data: { stock: { decrement: qty } } })` and check `count`.
2. **Pessimistic lock:** `SELECT ... FOR UPDATE` inside a transaction (`$queryRaw`).
3. **Optimistic locking:** a `version` column. Update `WHERE version = X`, and retry if it fails.
4. **DB constraint as backup:** `CHECK (stock >= 0)`.
5. **Reservation pattern (for payments):** reserve stock when the order is created (`reserved_until = now + 15 min`). Confirm it on payment and release it on failure/expiry. That way you never take money for an item you can't deliver. If a payment still arrives after the stock is gone → **auto-refund**.
6. For flash sales: a Redis `DECR` counter + a queue in front of the DB.

### 8. Storing payment status safely in DB
```prisma
enum PaymentStatus { CREATED AUTHORIZED CAPTURED FAILED REFUNDED PARTIALLY_REFUNDED }
model Payment {
  id                String        @id @default(cuid())
  orderId           String
  razorpayOrderId   String        @unique
  razorpayPaymentId String?       @unique
  amount            Int           // paise, never Float
  currency          String        @default("INR")
  status            PaymentStatus @default(CREATED)
  method            String?
  errorCode         String?
  rawResponse       Json?
  createdAt         DateTime      @default(now())
  updatedAt         DateTime      @updatedAt
  @@index([status, createdAt])
}
```
- **Amounts as integers (paise).** Enum statuses, **allowed transitions only** (a state machine), unique Razorpay IDs.
- Update only from **verified** sources (signature-checked verify call / webhook), inside a transaction.
- **Audit trail:** a `payment_events` table (append-only) holding every status change and the raw webhook payload.
- Never store card details (Razorpay handles PCI DSS). Keep secrets in env/Key Vault.
- Compare the amount and currency from Razorpay with your DB before marking it paid.

### 9. Testing Razorpay in test mode
- Use the **Test Mode** keys (`rzp_test_...`). No real money moves.
- Test cards from the Razorpay docs (e.g. success card numbers, failure scenarios), test UPI IDs `success@razorpay` / `failure@razorpay`, and test netbanking with success/failure buttons.
- **Webhooks locally:** expose localhost with **ngrok** / cloudflared, set up the webhook in Dashboard → Webhooks (test mode) with a secret, and use the dashboard's resend/test option.
- Test these cases: success, failure, user closes the popup, duplicate webhook, webhook arrives before `/verify`, refund, signature mismatch (should be rejected).
- Automated tests: mock the Razorpay SDK, and generate valid HMAC signatures in unit tests for the verify/webhook handlers.

### 10. Subscriptions / recurring payments
1. Create a **Plan** (`plans.create`: amount, period `monthly`, interval).
2. Create a **Subscription** (`subscriptions.create({ plan_id, customer_notify: 1, total_count: 12 })`) and save its `subscription_id` against the user.
3. Open Checkout with `subscription_id`. The user authorizes the **mandate** (card / UPI AutoPay / eMandate).
4. Verify the signature: HMAC of `razorpay_payment_id|subscription_id`.
5. Razorpay charges automatically each cycle. Handle these webhooks: `subscription.activated`, `subscription.charged` (extend access + save the invoice), `subscription.pending`/`halted` (payment failed → grace period → restrict access), `subscription.cancelled`, `subscription.completed`.
6. Access control: `user.subscriptionStatus` + `currentPeriodEnd`, updated **only from webhooks**.
7. Upgrades/downgrades with `subscriptions.update`. Cancel at the end of the cycle (`cancel_at_cycle_end`).

