# Section 3: Node + Microservices (12 Qs)

> [← Back to index](README.md)


### 1. Explain your microservices architecture (draw it)
```
                ┌────────────┐
 Client (Next) ─►  API Gateway (auth, rate limit, routing)  │
                └─────┬──────┘
        ┌─────────────┼───────────────┬────────────────┐
        ▼             ▼               ▼                ▼
   Auth Service   Order Service   Payment Service   Notification/Realtime
   (users, JWT)   (cart, orders)  (Razorpay, webhooks)  (Socket.io, email)
        │             │               │                ▲
     Postgres      Postgres        Postgres            │
                      │               │                │
                      └──── Redis / BullMQ (or Kafka) ─┘   ← events: order.created, payment.captured
```
**How to say it:** "Each service owns its own DB (no shared tables). Synchronous calls go over REST through the gateway. Business events like `payment.captured` go on a queue, so Order and Notification react asynchronously. Redis is used for caching, the Socket.io adapter and BullMQ."
*(Adjust this to what you actually built. If you built a modular monolith, say so and explain how you'd split it.)*

### 2. Monolith vs Microservices — why split?
- **Monolith:** one codebase and one deploy. Simple, fast to build, easy transactions. **Start here.**
- **Microservices:** independent deploys, scaling and tech per service, plus fault isolation. They cost more in network calls, distributed transactions, DevOps and debugging.
- **Reasons I split:** (1) Payment needed **independent scaling and isolation**, so a bug in notifications can't break payments. (2) Different release speed. (3) Webhooks and background jobs needed to scale separately.
- **Mature answer:** "I wouldn't start with microservices for a small team. I'd use a modular monolith and split once the boundaries are clear."

### 3. Communication: REST vs Queue vs gRPC
| | Type | Use when |
|---|---|---|
| **REST/HTTP** | Sync, request-response | You need an answer *now* (get user, validate cart) |
| **gRPC** | Sync, binary (protobuf), HTTP/2, typed contracts | Fast internal service-to-service calls, streaming |
| **Queue (BullMQ/RabbitMQ/Kafka)** | Async, event-driven | Fire-and-forget, retries, decoupling: send email, update inventory after payment |

**Rule:** use sync calls only when the user is waiting for the result. Everything else becomes an event, so one slow service doesn't slow down the whole chain.

### 4. API Gateway + Rate limiting
**API Gateway** is the single entry point. It handles routing, auth verification, rate limiting, CORS, request logging, SSL termination and response aggregation (examples: Nginx, Kong, AWS API Gateway, Azure APIM, or a custom Express gateway with `http-proxy-middleware`).

**Rate limiting** = limiting requests per user/IP per time window, to stop abuse, brute force and DDoS-like spikes.
- Algorithms: **fixed window**, **sliding window**, **token bucket** (allows short bursts).
- With many servers, store the counters in **Redis** so all instances share them.
```js
app.use('/api/auth/login', rateLimit({ windowMs: 15 * 60_000, max: 5, store: new RedisStore({ ... }) }));
```
Return **429 Too Many Requests** with a `Retry-After` header.

### 5. JWT + Refresh Token flow
1. Login → server issues an **access token** (short, 15 min) and a **refresh token** (long, 7–30 days).
2. Both go in **httpOnly Secure SameSite cookies**. Store a **hash** of the refresh token in the DB/Redis.
3. Each API call checks the access token (signature + expiry). No DB hit needed.
4. Access token expires → you get a 401 → the client calls `/auth/refresh` → the server checks the refresh token against the DB and issues **new access + new refresh tokens** (**rotation**) and invalidates the old one.
5. **Reuse detection:** if an old refresh token is used again, someone stole it, so revoke all sessions for that user.
6. Logout → delete the refresh token from the DB and clear the cookies.

**Why two tokens?** A stateless, fast access token, plus the ability to revoke via the refresh token.

### 6. Idempotency — why important for payments?
**Idempotent** = doing the same operation many times gives the **same result as doing it once**.
Networks retry, users double-click "Pay", and webhooks are delivered multiple times. Without idempotency you get **double charges and double orders**.
**How:**
- The client sends an `Idempotency-Key` (UUID) header. The server stores `key → response` (Redis/DB with a unique constraint). If the same key comes again, it returns the saved response and does not process again.
- DB **unique constraints** (e.g. `UNIQUE(razorpay_payment_id)`) are the final safety net.
- State checks: `UPDATE orders SET status='PAID' WHERE id=$1 AND status='PENDING'`.

### 7. Handling service failure: Circuit breaker, Retry
- **Timeout:** never wait forever (e.g. 3 s on internal calls).
- **Retry with exponential backoff + jitter:** 200 ms → 400 ms → 800 ms… Only for **transient** errors (5xx, timeouts) and only for **idempotent** operations.
- **Circuit breaker** (library: `opossum`): after N failures the circuit **opens**, and calls fail fast (or return a fallback) for some time. Then it goes **half-open** and tries a few calls. If they succeed it **closes** again. This protects the failing service and your threads.
- **Fallbacks:** cached data, a default response, "try again later".
- **Queue + DLQ:** failed jobs retry automatically, and after max attempts they go to a dead-letter queue for manual review.
- **Bulkhead:** separate pools so one slow dependency can't consume all resources.

### 8. Express middleware — how it works
Middleware is a function `(req, res, next)` that runs **in order** in a pipeline. Each one can modify `req`/`res`, end the response, or call `next()` to pass control on.
```js
app.use(helmet());
app.use(express.json());
app.use(requestLogger);
app.use('/api/orders', authenticate, ordersRouter);
app.use(notFound);
app.use((err, req, res, next) => {          // error middleware has 4 args
  logger.error(err);
  res.status(err.status || 500).json({ message: err.expose ? err.message : 'Internal error' });
});
```
- `next(err)` skips to the error handler. Order matters: the raw-body parser for webhooks must come **before** `express.json()` on that route.
- In Express 4, async errors need `try/catch` or a wrapper. Express 5 forwards rejected promises automatically.

### 9. File uploads + validation
- Use **Multer** (or Busboy) for `multipart/form-data`. Set `limits: { fileSize: 5 * 1024 * 1024 }`.
- **Validate:** whitelist MIME types, **check magic bytes** (`file-type` package), because the extension/MIME can be faked. Rename files with a UUID and never use the user's filename in a path.
- **Better at scale:** **presigned URLs** (S3 / Azure Blob SAS). The client uploads directly to storage, so the Node server doesn't carry the file bytes. The server stores only metadata.
- Extra: virus scan, image resize with `sharp` in a background job, private buckets + signed URLs for reading.

### 10. BullMQ vs Kafka
- **BullMQ:** a job queue on **Redis**. Supports retries, delays, priorities, cron jobs and concurrency. Great for **background tasks**: emails, PDF invoices, image processing, webhook processing, retrying failed payment verification.
- **Kafka:** a distributed **event log/stream**. Very high throughput, messages are kept (replay is possible), consumer groups, ordering per partition. Good for **event-driven microservices at scale**, analytics, audit logs and many consumers of the same event.
- **Answer:** "For my scale BullMQ is enough and simple: one job goes to one worker. I'd move to Kafka when multiple services need the same events, with replay and millions of events per day."

### 11. Securing a Node API
- **Helmet:** secure HTTP headers (CSP, HSTS, X-Frame-Options…).
- **CORS:** whitelist exact origins, `credentials: true` only for your own frontend. Never `*` with cookies.
- **Validation:** Zod/Joi on every body/query/param. Reject unknown fields.
- **SQL injection:** Prisma/parameterized queries. Be careful with `$queryRawUnsafe`.
- **Auth:** bcrypt/argon2 for passwords, short JWT, RBAC checks, and **object-level authorization** (does this order belong to you?).
- Rate limiting, request body size limit, HTTPS, secrets in env/Key Vault (never in git), `npm audit`/Dependabot, no stack traces sent in production responses, CSRF protection when using cookies (SameSite + CSRF token).

### 12. Logging + monitoring in production
- **Structured JSON logs** with **Pino** or Winston: level, timestamp, `requestId`/`traceId`, `userId`. No passwords, tokens or card data.
- A **correlation ID** passed across services (header `x-request-id`) so you can trace one request through all of them.
- Centralize the logs: ELK, Grafana Loki, Azure Monitor / App Insights, Datadog.
- **Metrics:** latency (p95/p99), error rate, throughput, queue length, DB pool usage (Prometheus + Grafana).
- **Tracing:** OpenTelemetry. **Errors:** Sentry. **Alerts:** Slack/PagerDuty when error rate or payment failures spike.
- Health endpoints `/health` and `/ready` for load balancers/Kubernetes.

