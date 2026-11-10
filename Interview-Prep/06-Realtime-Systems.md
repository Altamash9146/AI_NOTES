# Section 6: Realtime Systems (8 Qs)

> [← Back to index](README.md)


### 1. WebSockets vs Polling vs SSE
| | Direction | Use for |
|---|---|---|
| **Short polling** | Client asks every N sec | Simple, rare updates (payment status check every 5 s as a fallback) |
| **Long polling** | Server holds the request until data arrives | Fallback when WS is blocked |
| **SSE** (Server-Sent Events) | **Server → client only**, over HTTP, auto-reconnect | AI token streaming, notifications, live scores, order status |
| **WebSocket** | **Two-way**, persistent | Chat, multiplayer, collaborative editing, live tracking with client events |

**Rule:** do you need the client to send data often? Use WebSocket. Only server pushes? SSE is simpler and works with HTTP infrastructure. Updates are rare? Polling is fine.

### 2. Socket.io Rooms, Namespaces, Reconnection
- **Namespace** = a separate channel/endpoint on one connection with its own middleware, e.g. `/chat`, `/admin`, `/tracking`.
- **Room** = a group inside a namespace that sockets join/leave. Use it to broadcast to a subset: `room:orderId`, `chat:roomId`, `user:userId` (for all of a user's tabs/devices).
```js
io.of('/chat').on('connection', (socket) => {
  socket.on('join', (roomId) => socket.join(roomId));
  socket.on('message', (msg) => io.of('/chat').to(msg.roomId).emit('message', msg));
});
```
- **Reconnection:** the client reconnects automatically with exponential backoff (`reconnectionDelay`, `reconnectionDelayMax`). After reconnecting you must **re-join rooms** (on the server's `connection` event, or the client `connect` event) and **fetch the messages you missed** (send `lastMessageId`). Socket.io v4.6+ has **connection state recovery**. Show a "Reconnecting…" UI.

### 3. Socket.io with 2+ servers — Redis Adapter
**Problem:** user A is connected to server 1 and user B to server 2. `io.to(room).emit()` on server 1 only reaches sockets on server 1.
**Fix:** `@socket.io/redis-adapter`. Each server publishes broadcasts to Redis Pub/Sub, and every server delivers them to its own local sockets.
```js
const pub = createClient({ url: REDIS_URL }); const sub = pub.duplicate();
await Promise.all([pub.connect(), sub.connect()]);
io.adapter(createAdapter(pub, sub));
```
- You also need **sticky sessions** at the load balancer (when using HTTP long-polling fallback), or force `transports: ['websocket']`.
- Other services (like a payment worker) can emit through `@socket.io/redis-emitter` without running a socket server.

### 4. Online / offline status
- On `connection`: add the socketId to a Redis set `online:{userId}` and broadcast `user:online` to friends/rooms.
- On `disconnect`: remove the socketId. **Only when the set is empty** (all tabs closed) mark the user offline. **Delay it ~5–10 s** so a page refresh doesn't flicker.
- **Heartbeat:** Socket.io pings automatically. You can also use a Redis key with a TTL (`presence:{userId}` with a 60 s TTL, refreshed by heartbeat), so a crashed server doesn't leave users "online" forever.
- Save `lastSeen` in Postgres on disconnect ("last seen 5 min ago").

### 5. Message ordering in realtime chat
- **Don't trust client timestamps** (clocks differ). The **server assigns the order**: a DB auto-increment ID / a per-room sequence number (`seq`) / a Postgres timestamp at insert.
- Flow: the client sends `{ tempId, text }` → the server saves it to the DB (gets `id`, `seq`, `createdAt`) → it emits to the room → the client replaces the temp message (optimistic UI) and **sorts by seq**.
- Detect gaps: if the client receives `seq 12` after `seq 10`, fetch the missing ones.
- Ordering is guaranteed per connection (TCP) but not across servers, which is why the server-side sequence matters. With Kafka, partition by `roomId` to keep per-room order.
- Message status: `sent → delivered → read`, using acks (`socket.emit('msg', data, (ack) => ...)`).

### 6. Scaling realtime to 10k users
10k concurrent connections is very doable. One well-tuned Node server can hold 10k–50k idle sockets.
- **Horizontal scaling:** multiple Socket.io instances + **Redis adapter** + a load balancer with sticky sessions/WebSocket-only.
- Raise OS limits (`ulimit` file descriptors), use the `websocket` transport only, and use `uWebSockets.js`/`ws` for extreme scale.
- **Keep socket servers thin:** no heavy work inside handlers. Push work to queues (BullMQ), and save messages asynchronously or in batches.
- **Reduce traffic:** emit only to rooms (never broadcast to everyone), throttle frequent events (location every 3–5 s, typing every 2 s), send small payloads.
- Redis for presence and session data. Postgres with proper indexes (`messages(room_id, created_at)`), and partition/archive old messages.
- **Monitor:** connection count, event rate, memory per instance, event-loop lag. Autoscale on those.
- Load test with Artillery / k6 (WebSocket scenarios).

### 7. WebRTC vs Socket.io
| | Socket.io | WebRTC |
|---|---|---|
| Path | Client ↔ **Server** ↔ Client | **Peer-to-peer** (browser ↔ browser) |
| Protocol | TCP (WebSocket) | UDP (low latency) |
| Best for | Chat, notifications, live status | **Audio/video calls**, screen share, P2P file transfer |
| Server needed? | Yes, all traffic goes through it | Only for **signaling** (often done *with* Socket.io) + STUN/TURN servers for NAT traversal |

They work together: Socket.io sends the WebRTC offer/answer/ICE candidates (signaling), and then the media flows P2P. For group calls, use an SFU (mediasoup, LiveKit).

### 8. Realtime + Auth — socket handshake auth
```js
// client
const socket = io(URL, { auth: { token: accessToken }, withCredentials: true });

// server — runs once per connection, before 'connection'
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token || parseCookie(socket.handshake.headers.cookie).access_token;
    socket.data.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch { next(new Error('UNAUTHORIZED')); }
});

// authorize every action too
socket.on('join', async (roomId) => {
  if (!(await isMember(socket.data.user.id, roomId))) return socket.emit('error', 'FORBIDDEN');
  socket.join(roomId);
});
```
- Don't put the token in the URL query (it ends up in logs). Use `auth` or cookies.
- **Token expiry:** the socket stays connected after the JWT expires. Either re-validate periodically, or disconnect on expiry and let the client reconnect with a fresh token (handle the `connect_error` "UNAUTHORIZED" event, then refresh and reconnect).
- Check CORS origin, validate every incoming payload (Zod), and rate limit events per socket.

