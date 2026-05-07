#  Topic 8: Long Polling vs SSE

## Theory

Before WebSockets became mainstream, and even today in cases where WebSockets are overkill, developers needed ways to get real-time or near-real-time data from server to client over plain HTTP. Two techniques dominate this space:

- **Long Polling** — client pretends HTTP is real-time by holding requests open
- **Server-Sent Events (SSE)** — HTTP/1.1 streaming where the server pushes events continuously over one persistent connection

Neither requires a protocol upgrade. Both run on standard HTTP. That is their biggest advantage — they work everywhere HTTP works, with no special infrastructure needed.

---

## Long Polling — Internals

**Regular (Short) Polling — the naive approach:**
```
t=0s:  Client → "Any new messages?" → Server → "No"
t=2s:  Client → "Any new messages?" → Server → "No"
t=4s:  Client → "Any new messages?" → Server → "No"
t=6s:  Client → "Any new messages?" → Server → "Yes → {message}"
t=8s:  Client → "Any new messages?" → Server → "No"
```
- Simple but extremely wasteful
- Server gets hammered with requests even when nothing is happening
- Update latency = polling interval (2 seconds in the example above)

---

**Long Polling — the smarter version:**
```
t=0s:   Client → "Any new messages?" → Server holds connection open...
                                        ...
                                        ...  (server waits, no response yet)
                                        ...
t=6s:   New message arrives on server
        Server → "Yes → {message}" → Client
t=6s:   Client immediately opens next long poll request
t=6s:   Client → "Any new messages?" → Server holds connection open...
```

**How it works step by step:**
1. Client sends a request to the server
2. Server does **not** respond immediately if there is nothing to send
3. Server holds the connection open — typically up to 30–60 seconds (the timeout window)
4. When new data arrives, the server responds immediately with that data
5. Client receives the response, processes it, and **immediately** fires another long poll request
6. If timeout is reached with no data, server sends an empty response — client reconnects

**Key characteristics:**
- Still HTTP request-response — just with a delayed response
- Update latency is near-zero — server responds the moment data is ready
- Each update cycle requires a new HTTP request — headers sent every time
- Under high concurrency, server holds thousands of open connections consuming threads/file descriptors
- Works everywhere — no special browser or server support needed

---

## Server-Sent Events (SSE) — Internals

SSE is a proper HTTP streaming mechanism. The server opens one HTTP response and **never closes it** — it keeps writing events down the wire as they happen.

**The HTTP exchange:**

```
Client request:
GET /api/orders/o-789/events HTTP/1.1
Host: api.shopsphere.com
Accept: text/event-stream

Server response:
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"status": "CONFIRMED"}

data: {"status": "PREPARING"}

id: 3
event: statusUpdate
data: {"status": "SHIPPED", "eta": "Tomorrow 2PM"}

data: {"status": "DELIVERED"}
```

**SSE Event Format:**
Each event is plain text with a specific structure:
```
id: <event-id>           ← optional, used for reconnection
event: <event-type>      ← optional custom event name
data: <payload>          ← required, the actual data
                         ← blank line signals end of this event
```

- `data:` can span multiple lines — just repeat the field
- `id:` is the last event ID — browser stores this and sends it on reconnect via `Last-Event-ID` header
- `retry:` tells the browser how many milliseconds to wait before reconnecting after a disconnect

**Built-in Automatic Reconnection:**
```
Server goes down at t=30s

Browser automatically reconnects at t=33s (default 3s retry)
Sends: Last-Event-ID: 42

Server resumes stream from event 43 onward — no missed events
```

This is built into the browser's `EventSource` API — you get reconnection for free.

**SSE on HTTP/2:**
- On HTTP/1.1, each SSE stream occupies one TCP connection — 6 SSE streams = 6 connections
- On HTTP/2, multiple SSE streams multiplex over a single TCP connection — dramatically more efficient
- This is one reason SSE has seen a resurgence — HTTP/2 removes its main scaling concern on the client side

---

## Long Polling vs SSE — Side by Side

| Feature | Long Polling | SSE |
|---|---|---|
| Protocol | Standard HTTP | Standard HTTP (streaming) |
| Direction | Server → Client only | Server → Client only |
| Connection | New request per update | Single persistent connection |
| Overhead | Full HTTP headers per update | Headers only once on connect |
| Reconnection | Manual (client reopens request) | Automatic (built into EventSource) |
| Last event tracking | Manual | Built-in via `id` + `Last-Event-ID` |
| Browser support | Universal | Universal (except IE) |
| Proxy/firewall compatibility | Excellent | Good (some proxies buffer streams) |
| Complexity | Low | Low |
| Server resource usage | Higher (one thread per waiting request) | Lower (one streaming response per client) |

---

## When to Use What — Decision Framework

```
Do you need the CLIENT to send data continuously?
    └─ Yes → WebSocket
    └─ No  →
            Is update latency critical (sub-second)?
                └─ Yes, and high frequency → SSE
                └─ Near-real-time is fine → Long Polling or SSE
                
            Do you need to work through aggressive proxies / corporate firewalls?
                └─ Yes → Long Polling (most compatible)
                └─ No  → SSE (cleaner, less overhead)
                
            Do you need ordered, resumable streams with missed-event recovery?
                └─ Yes → SSE (built-in id + Last-Event-ID)
                └─ No  → Long Polling
```

**Practical summary:**
- **Long Polling** — maximum compatibility, simple to implement, acceptable when updates are infrequent
- **SSE** — cleaner, lower overhead, built-in reconnection, ideal for continuous server-push streams
- **WebSocket** — when client also sends data continuously or you need sub-10ms latency
- **Short Polling** — almost never, unless the data changes very infrequently and real-time does not matter

---

## Real-World Example — ShopSphere

**Long Polling — Admin Dashboard Order Count:**

Your admin panel shows a live count of pending orders. It updates every 30 seconds at most and the admin page needs to work on corporate networks that block WebSocket upgrades.

```
Admin browser → GET /api/admin/pending-orders/count (hold for up to 30s)
Order Service  → waits...
New order arrives
Order Service  → 200 OK { "count": 47 }
Admin browser  → immediately re-polls
```

Simple, compatible, no special infrastructure.

---

**SSE — Live Order Status Page:**

Customer is on the order tracking page watching their delivery status update.

```java
// Spring Boot SSE endpoint
@GetMapping(value = "/orders/{orderId}/status-stream",
            produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamOrderStatus(@PathVariable String orderId) {
    SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);
    
    orderEventService.subscribe(orderId, event -> {
        try {
            emitter.send(SseEmitter.event()
                .id(event.getId())
                .name("statusUpdate")
                .data(event.getPayload()));
        } catch (IOException e) {
            emitter.completeWithError(e);
        }
    });
    
    return emitter;
}
```

```javascript
// Frontend
const source = new EventSource('/api/orders/o-789/status-stream');

source.addEventListener('statusUpdate', (event) => {
    const data = JSON.parse(event.data);
    updateOrderStatusUI(data.status);
});

source.onerror = () => {
    // EventSource handles reconnection automatically
    console.log('Reconnecting...');
};
```

The browser's `EventSource` API handles all reconnection automatically. If the server restarts, the browser reconnects and sends `Last-Event-ID` so the server can resume from where it left off.

---

## How These Fit Into the Bigger Picture

```
Frequency of updates   |  Low              |  Medium           |  High (continuous)
Client also sends?     |  No               |  No               |  Yes
──────────────────────────────────────────────────────────────────────────────
Best approach          | Short/Long Poll   |  SSE              |  WebSocket
Example in ShopSphere  | Admin stats page  |  Order tracking   |  Live chat / collab
```

---

## Interview Q&A

**Q: What is the difference between short polling and long polling?**
Short polling sends a request every fixed interval regardless of whether new data exists — wasteful and high latency. Long polling holds the request open on the server until data is available or a timeout is reached, then responds immediately. Update latency drops to near-zero and unnecessary empty responses are eliminated.

**Q: What is SSE and how does it differ from long polling?**
SSE opens a single persistent HTTP connection over which the server streams events continuously. Long polling requires a new HTTP request for every update cycle — full headers are resent each time. SSE is more efficient, has built-in reconnection via the `EventSource` API and `Last-Event-ID` tracking, and is cleaner to implement. Long polling has better compatibility with aggressive proxies and firewalls that may buffer SSE streams.

**Q: How does SSE handle reconnection after a network drop?**
The browser's `EventSource` API handles reconnection automatically. Each SSE event can carry an `id` field. The browser stores the last received `id` and on reconnect sends it as the `Last-Event-ID` request header. The server can use this to resume the stream from the correct position, ensuring no events are missed.

**Q: Why would you choose long polling over SSE in production?**
When compatibility is the priority — some corporate proxies, CDNs, and older load balancers buffer HTTP streaming responses, which breaks SSE. Long polling uses standard request-response HTTP and is universally compatible. You would also prefer long polling when updates are infrequent — if events come every 30 seconds, the overhead of maintaining a persistent SSE connection is not worth it.

**Q: Can SSE replace WebSockets?**
For server-to-client only scenarios — yes, often SSE is the better choice. It is simpler, runs on HTTP/2 natively, and has built-in reconnection. WebSockets are necessary when the client needs to send a continuous stream of data to the server — chat messages, game input, collaborative edits. If the client only occasionally sends data, SSE for server-push combined with regular POST requests for client-to-server is a perfectly clean architecture.

---

Say **"next"** when ready for Topic 9 — Latency vs Throughput.
