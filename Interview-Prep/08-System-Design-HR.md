# Section 8: System Design + HR (10 Qs)

> [← Back to index](README.md)


> **System design answer structure (use every time):**
> 1. Clarify requirements (functional + non-functional: scale, latency, consistency) → 2. Rough numbers → 3. High-level diagram → 4. DB schema → 5. APIs → 6. Deep dive (the hard part) → 7. Scaling + failure handling → 8. Trade-offs.

### 1. Design a Realtime Order Tracking System (food-delivery style)
**Requirements:** place an order, restaurant accepts, a rider is assigned, the live rider location appears on a map, status updates (`PLACED → ACCEPTED → PREPARING → PICKED_UP → DELIVERED`), plus notifications.
**Components:**
```
Customer App ─┐                          ┌─► Order Service (Postgres) ──► events (Kafka/BullMQ)
Restaurant App├─► API Gateway ──────────►├─► Rider/Dispatch Service (Redis GEO for nearby riders)
Rider App ────┘   + Socket Gateway       ├─► Location Service (Redis: latest location; time-series for history)
                  (Socket.io + Redis     └─► Notification Service (push/SMS)
                   adapter)  ◄──────────── consumes events → emits to room order:{id}
```
- **Rider location:** the rider app sends GPS every 3–5 s → Location Service → Redis `GEOADD` + publish → the socket gateway emits to room `order:{id}`. **Don't write every ping to Postgres.** Batch it to a time-series store/cold storage.
- **Status updates:** Order Service updates the DB (a state machine, only valid transitions) → publishes `order.status_changed` → the socket gateway pushes it to the customer.
- **Dispatch:** `GEOSEARCH` for riders within 3 km → offer to the nearest → timeout → next rider.
- **ETA:** from a maps API, cached and recomputed every N seconds.
- **Scale/failure:** stateless services behind a load balancer, socket servers scaled with the Redis adapter, and polling fallback (`GET /orders/:id/status`) if the socket drops. Throttle location updates. On reconnect the client fetches the current state.

### 2. Design a Payment System with Razorpay + Microservices
- **Services:** Order Service, Payment Service, Inventory Service, Notification Service, connected by an event bus.
- **Flow:** Order created (PENDING, stock **reserved**) → Payment Service creates the Razorpay order, stored with `razorpay_order_id` → checkout → verify + **webhook** (source of truth) → Payment Service publishes `payment.captured` → Order → PAID, Inventory confirms the reservation, Notification sends the email/socket update.
- **Consistency across services:** the **Saga pattern** (each step has a compensation: payment failed → release stock; out of stock after payment → refund) + the **Outbox pattern** (write the DB change and the event to an `outbox` table in the same transaction, and a relay publishes it, so events are never lost).
- **Reliability:** idempotency keys, a unique `payment_id`, a webhook event table for dedupe, a reconciliation cron for stuck PENDING orders, a DLQ for failed events, retries with backoff.
- **Security:** HMAC verification, server-side amount, secrets in Key Vault, audit log, amounts in paise.
- **Trade-off line:** "Payments favour **consistency over availability**. I'd rather show 'processing' for a minute than mark something paid incorrectly."

### 3. Design an AI Chatbot with streaming + history
- **Frontend:** Next.js chat UI (`useChat`), streaming, a stop button, markdown rendering, a conversations sidebar.
- **Backend:** `/api/chat` route → auth → rate limit / quota check → load the conversation (last N messages + summary) → optional RAG retrieval (pgvector) → Azure OpenAI streaming → stream to the client over SSE → `onFinish` saves the assistant message + token usage.
- **DB:** `conversations`, `messages`, `usage` (per user per day), `documents/chunks` (RAG).
- **Scale:** stateless API (horizontal), Redis for rate limits and response cache, a queue for heavy jobs (document ingestion, summaries), multiple Azure deployments for TPM capacity.
- **Reliability/safety:** timeouts, retries, a fallback model, content filtering, prompt-injection defences, output validation, evals and logging (prompt version, latency, tokens, feedback 👍/👎).

### 4. Design a URL Shortener (or a Chat App)
**URL Shortener**
- **APIs:** `POST /shorten { longUrl, customAlias?, expiresAt? } → { shortUrl }` · `GET /:code → 301/302 redirect`.
- **DB:** `urls(id BIGSERIAL, code VARCHAR(10) UNIQUE, long_url TEXT, user_id, expires_at, created_at)` + `clicks(code, ts, country, referrer)` (or async analytics).
- **Code generation:** **Base62 of an auto-increment ID** (no collisions, short: 7 chars ≈ 3.5 trillion) or random 7 chars + a unique check. Base62 IDs are guessable, so add a random offset or use a Snowflake-style ID if that matters.
- **Read-heavy (100:1):** Redis cache `code → longUrl`, a CDN/edge, a DB index on `code`. 301 lets browsers cache (fewer hits, fewer analytics) and 302 tracks every click.
- **Analytics:** push click events to a queue and write them in batches. Never slow down the redirect.
- **Extras:** rate limit the create endpoint, malicious URL scanning, TTL cleanup job.

**Chat App (if asked instead)**
- Tables: `users`, `rooms`, `room_members(user_id, room_id, role, last_read_message_id)`, `messages(id, room_id, sender_id, content, seq, created_at)` with index `(room_id, created_at)`.
- APIs: `GET /rooms`, `GET /rooms/:id/messages?cursor=`, `POST /rooms`. Sockets: `join`, `message:send`, `message:new`, `typing`, `read`.
- Socket.io + Redis adapter, server-side ordering (`seq`), unread counts from `last_read_message_id`, presence in Redis, push notifications for offline users through a queue.

### 5. Best project — 2-minute pitch (template, fill in your real numbers)
> "My best project is **[name]**, a **[what it is, e.g. an e-commerce/food-ordering platform]** built with **Next.js, Node microservices, Postgres/Prisma, Razorpay and Socket.io**.
> **Problem:** [who used it and what problem it solved].
> **My role:** I owned **[payments + realtime / the backend / the full feature end-to-end]**.
> **Architecture:** a Next.js frontend with SSR/ISR, an API gateway, separate Auth, Order and Payment services, Redis + BullMQ for background jobs, and Socket.io for live order status.
> **Hardest part:** making payments reliable. I added signature verification, idempotent webhook handling with an event table, and a reconciliation job for stuck payments. **Result:** [e.g. zero double-charges, payment mismatch cases dropped from X/week to ~0].
> **Impact:** [numbers: users, orders/day, page-load improvement, API latency p95 from X to Y ms].
> **If I rebuilt it:** I'd add the outbox pattern and better observability with OpenTelemetry from day one."

**Tip:** have 3 numbers ready. Interviewers remember numbers.

### 6. Biggest challenge — STAR format (example; use your real story)
- **Situation:** "Some customers were charged but their orders still showed 'Pending'."
- **Task:** "I had to find the root cause and make sure no paid order was ever lost."
- **Action:** "From the logs I found the frontend `/verify` call failed when users closed the tab, and our webhook handler was failing signature checks because `express.json()` changed the raw body. I fixed it with `express.raw` on the webhook route, added an idempotent webhook-events table, moved processing to a BullMQ queue so we return 200 fast, and added a cron that reconciles PENDING orders older than 10 minutes using Razorpay's fetch API. I also added Slack alerts."
- **Result:** "Stuck orders went to zero, support tickets about payments dropped by [X%], and the reconciliation job became our safety net."

### 7. Why remote? Why leave the current job?
> "I've grown a lot in my current role. I've built [payments, realtime, AI features] end-to-end. Now I want to work on **larger scale and more complex systems** with a strong engineering team, and remote companies often have exactly that ownership culture. I'm productive remotely: I communicate in writing, write clear PR descriptions and docs, and I'm used to async work. Remote also lets me work with good teams anywhere, not just in my city."

**Never** speak badly of your current company. Frame it as *growth, scale, learning*. Money can be a reason, but don't make it the first one.

### 8. Current CTC + Expected
> "My current CTC is **7.2 LPA** (around **51k in-hand** per month). Based on my skills in Next.js, Node microservices, payments, realtime systems and Azure OpenAI, and the market for this role, I'm looking for **12–14 LPA**. I'm open to discussing the complete package."

**Tips to hold your number:**
- Say the range confidently, then **stop talking**. Don't justify too much.
- If they say "that's a 70–90% hike": *"I understand. My expectation is based on the role's scope and the market rate for these skills, not on my current salary. My current pay reflects my first role, not my current skill level."*
- If they push for one number, say **13–14**, not 12. Negotiation usually moves down.
- Ask about the fixed vs variable split, and benefits.

### 9. Notice period
> "My notice period is **30 days**. I can try to negotiate an early release of **15–20 days** if needed."

(Don't promise less than you can actually deliver.)

### 10. Where do you see yourself in 1 year?
> "In one year I want to be someone the team trusts to **own a complete feature or service end-to-end**, from design to production and monitoring, especially in areas like payments, realtime or AI. I want to deepen my system design skills, mentor juniors where I can, and have measurable impact on the product. Long term I'm aiming for a senior engineer role."

