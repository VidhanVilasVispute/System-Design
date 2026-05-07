# Topic 7: WebSockets

## Theory

HTTP follows a strict **request-response** model — the client asks, the server answers, conversation over. But what if the server needs to push data to the client without being asked? What if data is changing continuously and the client needs updates in real time?

This is where **WebSockets** come in.

A WebSocket is a **persistent, bidirectional communication channel** between client and server over a single TCP connection. Once opened, either side can send data to the other at any time — no request needed.

**The core difference:**
```
HTTP (Request-Response):
  Client ---request---> Server
  Client <--response--- Server
  (connection closes or idles)

WebSocket (Persistent Bidirectional):
  Client <----open connection----> Server
  Client <-------message---------> Server   (either side, any time)
  Client <-------message---------> Server
  Client <-------message---------> Server
  (connection stays open until explicitly closed)
```

**When do you need this?**
Whenever the server has information the client needs **as soon as it happens**, not when the client next asks:
- Live chat messages
- Real-time order status updates
- Live sports scores or stock prices
- Multiplayer game state
- Collaborative document editing (Google Docs style)
- Live notifications

---

## Internals — How WebSocket Connection Is Established

WebSockets start life as an HTTP request. This is called the **WebSocket Handshake** — it uses HTTP Upgrade to switch protocols:

**Step 1 — Client sends HTTP Upgrade request:**
```
GET /ws/notifications HTTP/1.1
Host: api.shopsphere.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**Step 2 — Server agrees to upgrade:**
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`101 Switching Protocols` — the only time you will ever see this status code. After this response, HTTP is done. The TCP connection is now a WebSocket tunnel — raw frames flow in both directions.

**Step 3 — WebSocket frames flow:**
```
Client --> Server:  { "type": "subscribe", "topic": "order:o-789" }
Server --> Client:  { "type": "update", "orderId": "o-789", "status": "SHIPPED" }
Server --> Client:  { "type": "update", "orderId": "o-789", "status": "DELIVERED" }
```

**WebSocket Frame Structure:**
- Each message is sent as one or more **frames**
- Frames have a small header (2–10 bytes) — much lighter than HTTP headers
- Frames can carry text (UTF-8) or binary data
- Special frame types: `ping`, `pong` (keepalive), `close`

**Keepalive — Ping/Pong:**
- Network infrastructure (load balancers, NAT gateways) closes idle connections after a timeout
- WebSocket has a built-in ping/pong mechanism — server sends `ping`, client responds `pong`
- This keeps the connection alive and lets both sides detect dead connections

---

## WebSocket vs HTTP Polling Alternatives

Before WebSockets, developers used workarounds to simulate real-time:

**Short Polling:**
```
Client: "Any updates?" → Server: "No"    (every 2 seconds)
Client: "Any updates?" → Server: "No"
Client: "Any updates?" → Server: "Yes, here it is"
```
- Simple but wasteful — hammers the server with requests even when nothing has changed
- High latency — update only arrives at next poll interval

**Long Polling:**
```
Client: "Any updates?" → Server holds the connection open...
                          ...until an update occurs or timeout
                        → Server: "Yes, here's the update"
Client: immediately opens a new long-poll request
```
- Better than short polling — server only responds when data is ready
- Still HTTP — each update requires a new request-response cycle
- Double the connections under load — one waiting, one being processed

**Server-Sent Events (SSE):**
```
Client ---request---> Server
Client <==stream====  Server  (server pushes events continuously)
```
- Server can push multiple events over a single HTTP connection
- **Unidirectional** — only server → client
- Simple, works over HTTP/2 natively, automatic reconnection built in
- Good for dashboards, live feeds, notifications where client never needs to send data back

**WebSocket:**
- **Bidirectional** — either side can send anytime
- Lower overhead per message than any HTTP-based approach
- More complex to scale (stateful connections)
- Use when the client also needs to send data continuously (chat, gaming, collaborative editing)

---

## Scaling WebSockets — The Hard Part

WebSockets are **stateful** — a connection is pinned to a specific server instance. This breaks horizontal scaling.

**The problem:**
```
User A connected to Server 1
User B connected to Server 2

User A sends message to User B
Server 1 has no connection to User B
How does the message get delivered?
```

**Solution — Pub/Sub Backplane:**
```
Server 1  ←→  Redis Pub/Sub  ←→  Server 2
                    ↑
              Message bus shared
              by all WebSocket servers
```

- When User A sends a message, Server 1 publishes it to a Redis channel
- Server 2 is subscribed to that channel, receives the message, and pushes it to User B
- Every WebSocket server subscribes to all relevant channels
- Redis Pub/Sub or Kafka can act as this backplane

**Load Balancer Configuration:**
- Standard round-robin load balancing breaks WebSockets — subsequent requests might go to a different server than where the WebSocket connection was opened
- Requires **sticky sessions** (session affinity) — all requests from the same client go to the same server
- Or better — use a dedicated WebSocket service layer that handles all persistent connections

---

## Real-World Example — ShopSphere Live Order Tracking

**Scenario:** User places an order and watches its status update in real time — Confirmed → Preparing → Shipped → Delivered.

**Architecture:**
```
Browser <==WebSocket==> Notification Service <-- RabbitMQ <-- Order Service
                                           <-- RabbitMQ <-- Logistics Service
```

**Flow:**
1. User places order — browser opens WebSocket connection to Notification Service: `wss://api.shopsphere.com/ws`
2. Client sends: `{ "type": "subscribe", "orderId": "o-789" }`
3. Notification Service registers this connection against `order:o-789`
4. Order Service publishes status change events to RabbitMQ
5. Notification Service consumes from RabbitMQ and pushes directly to the subscribed WebSocket connection
6. Browser receives: `{ "type": "ORDER_UPDATE", "status": "SHIPPED", "eta": "Tomorrow 2PM" }`

**Why not polling here?**
- Polling every 2 seconds × 10,000 concurrent users = 5 million requests per minute hitting your Order Service for mostly empty responses
- WebSocket + event-driven push = zero unnecessary requests, updates arrive in milliseconds

---

## Interview Q&A

**Q: What is a WebSocket and how is it different from HTTP?**
HTTP is a request-response protocol — the client always initiates. WebSocket is a persistent bidirectional channel — once established, either side can send data at any time without a new request. It starts as an HTTP connection and upgrades via a 101 status code, after which the TCP connection becomes a WebSocket tunnel with minimal framing overhead.

**Q: When would you use WebSocket vs SSE?**
Use SSE when you only need server-to-client push — live feeds, dashboards, notifications — because it is simpler, works natively over HTTP/2, and has automatic reconnection. Use WebSocket when you need bidirectional communication — chat, collaborative editing, multiplayer gaming — where the client also sends a continuous stream of data to the server.

**Q: How do you scale WebSockets horizontally?**
WebSocket connections are stateful and pinned to a server instance, which breaks standard horizontal scaling. The solution is a shared pub/sub backplane — typically Redis Pub/Sub or Kafka. When a message needs to reach a client on a different server instance, it is published to the backplane and all server instances receive it, delivering it to the correct connected client. Load balancers must be configured with sticky sessions or session affinity.

**Q: What is the WebSocket handshake and why does it start with HTTP?**
WebSocket uses HTTP for its handshake because it needs to traverse the same ports (80/443) and infrastructure as HTTP without requiring new firewall rules. The client sends an HTTP Upgrade request, the server responds with 101 Switching Protocols, and the underlying TCP connection is repurposed as a WebSocket channel. This allows WebSockets to work transparently through existing web infrastructure.

**Q: How do you handle dropped WebSocket connections?**
WebSockets use ping/pong frames for keepalive — the server periodically sends a ping and expects a pong back. If no pong arrives within a timeout, the connection is considered dead and cleaned up. On the client side, most WebSocket libraries implement automatic reconnection with exponential backoff. The client must also re-send any subscription state after reconnecting since the server-side session is gone.

---

