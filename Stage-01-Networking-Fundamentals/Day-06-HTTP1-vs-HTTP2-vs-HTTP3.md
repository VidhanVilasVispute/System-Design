# Topic 6: HTTP/1.1 vs HTTP/2 vs HTTP/3

## Theory

HTTP has gone through three major versions, each solving a fundamental performance problem of the previous one. Understanding the evolution is important because:

- You will be asked about it in interviews
- It directly affects how you design APIs and configure infrastructure
- Each version makes different trade-offs that matter at scale

The core problem each version tries to solve is **latency** — how to get data from server to client faster, especially when there are many resources to transfer.

---

## HTTP/1.1 — The Baseline

Released in 1997, still widely used today.

**How it works:**
- One request at a time per TCP connection
- Client sends request, waits for response, then sends the next request
- **Keep-Alive** — introduced connection reuse, so the same TCP connection can handle multiple requests sequentially instead of opening a new one each time

**The core problem — Head-of-Line Blocking:**
```
TCP Connection 1:
  Request A -----> Response A -----> Request B -----> Response B

If Response A is slow, Request B is stuck waiting
```

**Workaround browsers used — Domain Sharding:**
- Open 6 parallel TCP connections per domain
- Download multiple resources simultaneously across those 6 connections
- Ugly hack — 6 TCP handshakes, 6 TLS handshakes, high memory overhead on the server

**Other problems:**
- **Verbose headers** — every request sends the full set of headers including cookies, User-Agent, Accept etc. — repeated identically on every single request with no compression
- **No prioritisation** — all requests are equal, you cannot say "load the CSS before the images"
- **No server push** — server can only respond, never proactively send

---

## HTTP/2 — Multiplexing Over TCP

Released in 2015. Built on Google's SPDY protocol.

**The core innovation — Multiplexing:**
```
Single TCP Connection:
  Stream 1: Request A ---------> Response A
  Stream 3: Request B ---> Response B
  Stream 5: Request C ----------------> Response C

All streams are interleaved over the same TCP connection simultaneously
```

- Multiple requests and responses fly in parallel over **one** TCP connection
- No need for domain sharding anymore
- Each stream has a unique ID and is independent

**Other improvements:**

**Header Compression (HPACK):**
- Maintains a shared compression table between client and server
- Repeated headers (like `Authorization`, `User-Agent`) are sent as index references instead of full strings
- Dramatically reduces header overhead on APIs with many small requests

**Stream Prioritisation:**
- Streams can be assigned weights and dependencies
- Tell the server — load CSS (priority 1) before images (priority 8)
- The server can allocate bandwidth accordingly

**Server Push:**
- Server can proactively send resources it knows the client will need
- e.g. Client requests `index.html` — server pushes `styles.css` and `app.js` before the client even asks
- In practice this was hard to use correctly and is being deprecated in HTTP/3

**Binary Protocol:**
- HTTP/1.1 is text-based — human readable but slower to parse
- HTTP/2 is binary — more compact, faster to parse, less error-prone

**The remaining problem — TCP Head-of-Line Blocking:**
```
HTTP/2 solved application-level HOL blocking
But TCP-level HOL blocking still exists:

If one TCP packet is lost, ALL streams on that connection stall
while TCP waits for retransmission — even streams that had nothing
to do with the lost packet
```

This is the fundamental limit of building on TCP.

---

## HTTP/3 — Built on QUIC (UDP)

Released officially in 2022. This is a ground-up redesign of the transport layer.

**The core decision — abandon TCP, build on UDP:**

Instead of fixing TCP, HTTP/3 runs on **QUIC** — a new transport protocol built on top of UDP that reimplements what TCP does well, but fixes what TCP does poorly.

**How QUIC solves TCP's problems:**

**Independent Streams:**
```
QUIC Connection:
  Stream 1: Packet lost → only Stream 1 pauses for retransmission
  Stream 3: Unaffected — keeps flowing
  Stream 5: Unaffected — keeps flowing

vs HTTP/2 over TCP:
  Any packet lost → ALL streams pause
```

QUIC implements streams at the transport layer. A lost packet only blocks the stream it belongs to — other streams are completely unaffected.

**0-RTT and 1-RTT Connection Setup:**
```
TCP + TLS 1.2:  3-way handshake (1.5 RTT) + TLS handshake (2 RTT) = 3.5 RTT before data
TCP + TLS 1.3:  3-way handshake (1 RTT) + TLS 1.3 (1 RTT) = 2 RTT before data
QUIC + TLS 1.3: Combined in 1 RTT for new connections, 0 RTT for resumed connections
```

QUIC combines the transport and TLS handshakes into a single round-trip. For resumed connections (client has seen this server before), data can flow immediately with **0-RTT**.

**Connection Migration:**
- TCP connections are tied to a 4-tuple: source IP, source port, destination IP, destination port
- If your phone switches from WiFi to 4G, your IP changes → TCP connection breaks → reconnect required
- QUIC connections use a **Connection ID** instead — independent of IP/port
- Switching networks is seamless, connection survives without interruption

**Built-in Encryption:**
- QUIC requires TLS 1.3 — encryption is mandatory, not optional
- More of the packet is encrypted compared to TCP (even packet numbers are encrypted)

---

## Side-by-Side Comparison

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No | Yes (streams) | Yes (independent streams) |
| HOL Blocking | Application + TCP | TCP only | None |
| Header Compression | None | HPACK | QPACK |
| Connection Setup | TCP + TLS separately | TCP + TLS separately | Combined (1-RTT / 0-RTT) |
| Server Push | No | Yes (deprecated) | Limited |
| Binary Protocol | No (text) | Yes | Yes |
| Connection Migration | No | No | Yes |
| Encryption | Optional (HTTPS) | Optional (HTTPS) | Mandatory |
| Adoption | Universal | ~65% of web | ~30% and growing |

---

## Real-World Example — ShopSphere

**HTTP/1.1 scenario — ShopSphere frontend loading:**
- Browser opens 6 parallel connections to `api.shopsphere.com`
- Loads product images, CSS, JS, API responses across those 6 connections
- If one API call is slow, it blocks one of the 6 slots

**HTTP/2 scenario — ShopSphere with Nginx + HTTP/2:**
```nginx
server {
    listen 443 ssl http2;
    server_name api.shopsphere.com;
    ...
}
```
- All API calls from the frontend multiplex over a single connection
- Header compression kicks in — `Authorization: Bearer <JWT>` is sent as an index after the first request
- Significant latency reduction for pages that make 10–15 API calls on load

**HTTP/3 scenario — ShopSphere on Cloudflare:**
- Enable HTTP/3 / QUIC on your Cloudflare CDN config with one toggle
- Mobile users switching between WiFi and 4G maintain their session without reconnecting
- Users on lossy mobile networks (high packet loss) see dramatically better performance because a dropped packet no longer stalls all streams

**Inter-service communication inside ShopSphere:**
- Internal microservice calls (Order Service → Payment Service) use HTTP/1.1 or HTTP/2
- gRPC (which runs on HTTP/2) is a popular choice for internal service communication because of multiplexing and binary protocol efficiency
- HTTP/3 is primarily a client-facing optimisation — internal traffic usually stays on HTTP/2

---

## Interview Q&A

**Q: What problem does HTTP/2 solve that HTTP/1.1 could not?**
HTTP/1.1 could only process one request at a time per connection, causing head-of-line blocking — a slow response blocks all subsequent requests. HTTP/2 introduces multiplexing, allowing multiple requests and responses to be in-flight simultaneously over a single TCP connection, eliminating application-level HOL blocking and removing the need for domain sharding hacks.

**Q: HTTP/2 has multiplexing — so why was HTTP/3 still needed?**
HTTP/2 solved application-level HOL blocking but TCP-level HOL blocking remained. If any TCP packet is lost, TCP's retransmission mechanism stalls all HTTP/2 streams on that connection — even ones unrelated to the lost packet. HTTP/3 runs on QUIC over UDP, where streams are independent at the transport layer — a lost packet only blocks the stream it belongs to, not others.

**Q: What is 0-RTT in HTTP/3 and what is the trade-off?**
0-RTT allows a client that has previously connected to a server to send data immediately on reconnection with zero round-trip setup time, because cryptographic parameters from the previous session are reused. The trade-off is **replay attacks** — a network attacker could capture and re-send the 0-RTT data. For this reason 0-RTT should only be used for idempotent requests like GETs, never for state-changing operations like payments.

**Q: Why does HTTP/3 use UDP when UDP is unreliable?**
UDP is unreliable by itself, but QUIC reimplements reliability on top of UDP — it has its own acknowledgement, retransmission, and ordering mechanisms. The reason for choosing UDP as the base is that UDP gives QUIC full control over these mechanisms, allowing it to implement them more efficiently than TCP does — particularly with independent stream-level retransmission instead of connection-level blocking.

**Q: How would you enable HTTP/2 for ShopSphere in production?**
HTTP/2 is typically enabled at the load balancer or reverse proxy layer — in Nginx with `listen 443 ssl http2`, or automatically by AWS ALB which supports HTTP/2 by default. The backend services themselves can still use HTTP/1.1 internally — HTTP/2 terminates at the edge. The requirement is that HTTPS must be enabled, as all major browsers only support HTTP/2 over TLS.

---
