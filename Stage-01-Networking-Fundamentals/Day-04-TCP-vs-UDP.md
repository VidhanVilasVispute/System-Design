# Topic 4: TCP vs UDP

## Theory

Once you have an IP address and a port, you need a **transport protocol** to actually move data between two machines. There are two options:

- **TCP (Transmission Control Protocol)** — reliable, ordered, guaranteed delivery. The postal service that confirms your package arrived.
- **UDP (User Datagram Protocol)** — fast, lightweight, no guarantees. A loudspeaker announcement — you broadcast it and don't care who received it.

Neither is better — they solve different problems. Choosing the wrong one is a real system design mistake.

---

## Internals — TCP

TCP is **connection-oriented**. Before any data flows, a connection must be established.

**3-Way Handshake — how a TCP connection is opened:**
```
Client          Server
  |--- SYN ------->|     "I want to connect, my seq number is X"
  |<-- SYN-ACK ----|     "OK, I acknowledge X, my seq number is Y"
  |--- ACK ------->|     "I acknowledge Y, connection established"
```

**4-Way Handshake — how a TCP connection is closed:**
```
Client          Server
  |--- FIN ------->|     "I'm done sending"
  |<-- ACK --------|     "Acknowledged"
  |<-- FIN --------|     "I'm done too"
  |--- ACK ------->|     "Acknowledged, connection closed"
```

**What TCP guarantees:**

- **Reliability** — every packet is acknowledged. If no ACK is received, the sender retransmits.
- **Ordering** — packets are numbered with sequence numbers. Even if they arrive out of order, TCP reassembles them in the correct order.
- **Error checking** — every TCP segment has a checksum. Corrupted packets are discarded and retransmitted.
- **Flow control** — the receiver advertises a **window size** — how much data it can buffer. The sender respects this so it doesn't overwhelm the receiver.
- **Congestion control** — TCP detects network congestion (via packet loss) and slows down automatically — algorithms like **slow start**, **AIMD (Additive Increase Multiplicative Decrease)**.

**Cost of all this:** latency. Every guarantee adds overhead — handshakes, ACKs, retransmissions, reordering buffers.

---

## Internals — UDP

UDP is **connectionless**. You just fire packets and move on.

```
Client                    Server
  |--- Datagram ---------->|    "Here's some data"
  |--- Datagram ---------->|    "Here's more data"
  |                             (no ACK, no handshake, no guarantee)
```

**What UDP provides:**
- No connection setup — just send immediately
- No retransmission — lost packet is gone forever
- No ordering — packets may arrive out of sequence
- No congestion control — you send at full speed regardless
- Very low overhead — just 8 bytes of header vs TCP's 20+ bytes

**When is this acceptable?**
When you care more about **speed and freshness** than completeness. A lost video frame is better re-rendered than waited for. A retransmitted GPS location that arrives late is useless.

---

## TCP vs UDP — Side by Side

| Feature | TCP | UDP |
|---|---|---|
| Connection | Required (3-way handshake) | None |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Guaranteed | Not guaranteed |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Error recovery | Yes — retransmission | No |
| Flow control | Yes | No |
| Header size | 20+ bytes | 8 bytes |
| Use cases | HTTP, SSH, DB connections | Video streaming, DNS, gaming |

---

## When to Use Each — The Real Decision

**Use TCP when:**
- Data integrity is non-negotiable — financial transactions, order placement, authentication
- You are building APIs — HTTP/HTTPS runs on TCP
- Database connections — PostgreSQL, MySQL, Redis all use TCP
- File transfers — losing a byte is unacceptable

**Use UDP when:**
- Real-time data where latency matters more than completeness — live video, voice calls, gaming
- You implement your own reliability at the application layer — QUIC does this (HTTP/3 is built on QUIC over UDP)
- Broadcasting to multiple recipients — UDP supports multicast, TCP does not
- DNS queries — small, single request-response, fast matters more than reliability; if it fails, just retry

---

## Real-World Example — ShopSphere

**TCP usage in ShopSphere:**
- Every HTTP call between your microservices — API Gateway → Order Service → Payment Service — uses TCP
- Your Spring Boot apps connect to PostgreSQL over TCP port 5432
- Redis connections on TCP port 6379
- RabbitMQ on TCP port 5672
- Kafka brokers communicate with producers and consumers over TCP port 9092

**Where UDP would appear in a ShopSphere-like system:**
- If you added a **live order tracking** feature with real-time map updates — UDP or WebRTC
- **Metrics and telemetry** — some monitoring agents use UDP-based protocols (StatsD uses UDP) because a dropped metric datapoint is acceptable
- **DNS resolution** — every time your services resolve hostnames internally, those DNS queries go over UDP

---

## Interview Q&A

**Q: What is the difference between TCP and UDP?**
TCP is connection-oriented and guarantees reliable, ordered delivery through handshakes, ACKs, and retransmissions. UDP is connectionless, has no delivery guarantees, but is much faster with minimal overhead. Use TCP when data integrity matters, UDP when speed and low latency matter more than completeness.

**Q: Why does HTTP use TCP and not UDP?**
HTTP transfers web pages, API responses, and files where every byte must arrive correctly and in order. A missing or reordered packet would corrupt the response. TCP's reliability guarantees make it the right choice. HTTP/3 is the exception — it runs on QUIC which is built on UDP but reimplements reliability at the application layer with better performance characteristics.

**Q: What happens when a TCP packet is lost?**
The sender waits for an ACK within a timeout window. When no ACK arrives, it retransmits the packet. The receiver's TCP stack holds later packets in a buffer and only delivers data to the application once the gap is filled and ordering is restored.

**Q: Why is UDP used for DNS?**
DNS queries are tiny single request-response exchanges. UDP is faster because there is no handshake overhead. If a DNS query is lost, the resolver simply retries — so reliability is handled at the application level. For large DNS responses (zone transfers), TCP is used instead.

**Q: What is the 3-way handshake and why does it matter in system design?**
It is the TCP connection establishment process — SYN, SYN-ACK, ACK. It matters in system design because every new TCP connection costs one round-trip before any data flows. This is why **connection pooling** exists — reusing existing connections avoids paying this cost on every database query or service call.

**Q: Can you build reliability on top of UDP?**
Yes — this is exactly what QUIC does. QUIC runs over UDP but implements its own packet acknowledgement, retransmission, and ordering at the application layer. It also multiplexes streams so a lost packet in one stream doesn't block others — solving TCP's head-of-line blocking problem. HTTP/3 is built entirely on QUIC.

---
