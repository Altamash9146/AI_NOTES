# Section 4: Postgres + Prisma (10 Qs)

> [← Back to index](README.md)


### 1. Why Postgres over Mongo for payments?
- **ACID transactions** across multiple tables (order + payment + inventory change together, or nothing changes).
- **Strong consistency + constraints:** foreign keys, `UNIQUE(payment_id)`, `CHECK (amount > 0)`. The DB itself blocks bad data.
- Relational data (users → orders → items → payments) fits tables naturally. JOINs and reporting are easy.
- Row-level locking (`SELECT ... FOR UPDATE`) for race conditions.
- `NUMERIC`/integer types for money (store **paise as integers**, never floats).
- Mongo does have transactions now, but money data is relational and needs strict schemas, so Postgres is the natural fit. I'd still use JSONB in Postgres for flexible data like webhook payloads.

### 2. Indexing — when to add? What is a B-Tree?
- An **index** is a separate sorted data structure that finds rows without scanning the whole table (like a book index).
- **B-Tree** (the Postgres default): a balanced tree. Lookups, range queries (`>`, `<`, `BETWEEN`) and `ORDER BY` take O(log n).
- **Add an index when** a column is used often in `WHERE`, `JOIN`, `ORDER BY`, and has high selectivity, e.g. `orders(user_id)`, `payments(razorpay_order_id)`, `messages(room_id, created_at)`.
- **Composite index** order matters: `(user_id, created_at)` helps `WHERE user_id = ? ORDER BY created_at`.
- **Cost:** slower writes and extra storage. Don't index everything or low-cardinality columns like boolean flags (use a partial index instead: `WHERE status = 'PENDING'`).
- Other types: GIN (JSONB, full-text), Hash, and pgvector indexes (HNSW) for embeddings.
```prisma
model Order { id String @id  userId String  createdAt DateTime @default(now())
  @@index([userId, createdAt]) }
```

### 3. N+1 problem in Prisma
**N+1** = 1 query to get N rows, then 1 extra query **for each row** to get related data. 100 orders means 101 queries.
```ts
// ❌ N+1
const orders = await prisma.order.findMany();
for (const o of orders) o.items = await prisma.orderItem.findMany({ where: { orderId: o.id } });

// ✅ fixed — Prisma batches the related data
const orders = await prisma.order.findMany({
  include: { items: { include: { product: true } } },
});
// ✅ even better — fetch only needed fields
const orders2 = await prisma.order.findMany({
  select: { id: true, total: true, items: { select: { qty: true, product: { select: { name: true } } } } },
});
```
Also: `relationLoadStrategy: 'join'` (a single SQL JOIN). In GraphQL, use DataLoader. Detect N+1 by turning on Prisma query logging (`log: ['query']`).

### 4. Transactions in Prisma — why needed for payment?
When payment succeeds, several things must happen **together**: mark the payment captured, mark the order paid, reduce stock, write a ledger entry. If one step fails halfway, you get money taken with no order, or negative stock.
```ts
await prisma.$transaction(async (tx) => {
  const payment = await tx.payment.update({ where: { razorpayOrderId }, data: { status: 'CAPTURED', razorpayPaymentId } });
  await tx.order.update({ where: { id: payment.orderId }, data: { status: 'PAID' } });
  for (const item of items) {
    const r = await tx.product.updateMany({
      where: { id: item.productId, stock: { gte: item.qty } },
      data: { stock: { decrement: item.qty } },
    });
    if (r.count === 0) throw new Error('OUT_OF_STOCK');   // rolls back everything
  }
}, { isolationLevel: 'Serializable', timeout: 10_000 });
```
- Two forms: **interactive** (callback, as above) and **batch** `prisma.$transaction([q1, q2])`.
- Keep transactions **short**: never call Razorpay or other HTTP APIs inside one, because it holds locks.

### 5. Relations in Prisma schema
```prisma
// One-to-One: User ↔ Profile
model User {
  id      String   @id @default(cuid())
  email   String   @unique
  profile Profile?
  orders  Order[]           // One-to-Many
  rooms   RoomMember[]      // Many-to-Many (explicit)
}
model Profile { id String @id @default(cuid())  userId String @unique  user User @relation(fields: [userId], references: [id]) }

// One-to-Many: User → Orders
model Order { id String @id @default(cuid())  userId String  user User @relation(fields: [userId], references: [id])  @@index([userId]) }

// Many-to-Many: User ↔ Room (explicit join table so extra fields like role/joinedAt can be stored)
model Room { id String @id @default(cuid())  members RoomMember[] }
model RoomMember {
  userId   String
  roomId   String
  role     String   @default("member")
  joinedAt DateTime @default(now())
  user User @relation(fields: [userId], references: [id])
  room Room @relation(fields: [roomId], references: [id])
  @@id([userId, roomId])
}
```
- 1–1 = foreign key + `@unique`. 1–N = foreign key on the "many" side. N–N = join table (implicit `Post[]`↔`Tag[]` works too, but explicit is better for extra fields).

### 6. Pagination: Offset vs Cursor
| | Offset (`skip/take`) | Cursor |
|---|---|---|
| Query | `LIMIT 20 OFFSET 10000` | `WHERE id < lastId ORDER BY id DESC LIMIT 20` |
| Speed on deep pages | slow (the DB still reads and skips 10,000 rows) | fast (uses the index) |
| Jump to page 50 | ✅ | ❌ |
| Data changes while scrolling | duplicates/missed rows | stable |
| Use for | Admin tables with page numbers | Infinite scroll, chat history, feeds |
```ts
prisma.message.findMany({ take: 20, skip: 1, cursor: { id: lastId }, orderBy: { createdAt: 'desc' }, where: { roomId } });
```

### 7. ACID with a payment example
- **Atomicity:** the debit from A and the credit to B both happen or neither does. No "money disappeared".
- **Consistency:** constraints always hold (balance ≥ 0, every payment has a valid order).
- **Isolation:** two parallel payments don't see each other's half-done work (no double-spend of the same balance or stock). Postgres levels: Read Committed (default), Repeatable Read, Serializable.
- **Durability:** once COMMIT returns "paid", it survives a crash or power loss (WAL, the write-ahead log).

### 8. Optimizing a slow query — EXPLAIN ANALYZE
1. Find the slow queries: `pg_stat_statements`, Prisma query logs, APM.
2. Run `EXPLAIN ANALYZE SELECT ...` and read the plan:
   - **Seq Scan** on a big table → probably a missing index.
   - **Estimated rows vs actual rows** very different → run `ANALYZE` (stale statistics).
   - Expensive **Sort** / **Nested Loop** on large sets → index the sort/join columns.
3. Fixes: add a proper (composite/partial) index, `SELECT` only the needed columns, avoid `LIKE '%x%'` (use a trigram/full-text index), avoid functions on indexed columns (`WHERE LOWER(email)` → expression index), cursor pagination, cache hot reads in Redis, materialized views for reports.
4. Run EXPLAIN ANALYZE again and compare the time.

### 9. Prisma migrations in production
- Dev: `prisma migrate dev` creates the SQL migration files. **Commit them to git.**
- Prod/CI: **`prisma migrate deploy`** only. Never `migrate dev` or `db push` against production.
- Run migrations as a separate CI/CD step **before** the new code starts (or as a release job).
- **Zero-downtime (expand → contract):** (1) add the new nullable column, (2) deploy code that writes to both columns, (3) backfill data in batches, (4) switch reads, (5) drop the old column in a later release. Never rename or drop in one step.
- Review the generated SQL (big tables: create indexes `CONCURRENTLY`, which may need a custom SQL migration). Back up before risky migrations. `prisma migrate resolve` fixes a failed migration state.

### 10. Connection pooling — what and why?
- Opening a Postgres connection is expensive (a process per connection, ~10 MB), and Postgres has a max connection limit (often 100).
- A **pool** keeps a set of open connections and reuses them across requests.
- **Problem:** 10 Node instances × 10 connections, or serverless functions each opening their own, quickly gives "too many connections".
- **Solution:** set `connection_limit` in the Prisma URL, use **PgBouncer** (transaction mode, add `?pgbouncer=true`) or Prisma Accelerate / a managed pooler. Use a **single PrismaClient instance** (a global singleton in Next.js dev, to avoid a new client on every hot reload).

