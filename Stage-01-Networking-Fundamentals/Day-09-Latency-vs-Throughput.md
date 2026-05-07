# Topic 9: Latency vs Throughput

## Theory

Latency and throughput are the two most fundamental ways to measure performance in any system. They are related but completely different things, and optimising for one can actively hurt the other. Confusing them in an interview is a red flag — let's make sure that never happens.

**Latency** — how long a single operation takes from start to finish. Time. Duration. Delay.
- "How long did it take for my request to get a response?"
- Measured in milliseconds, microseconds, seconds

**Throughput** — how many operations a system can handle per unit of time. Volume. Rate. Capacity.
- "How many requests can this system process per second?"
- Measured in requests/second, transactions/second, MB/second

**The simple analogy:**
```
Imagine a highway:

Latency   = how long it takes one car to drive from city A to city B
Throughput = how many cars per hour pass through the highway

A wide 10-lane highway has high throughput but if there's traffic,
each car still takes a long time — high latency.

A single-lane empty road has low throughput but a car flies through
in minutes — low latency.
```

---

## Internals — Understanding Latency

Latency is not one number — it is a **distribution**. This is critical to understand.

**Components of latency in a network request:**

```
Total Latency = Network latency + Processing latency + Queuing latency

Network latency:
  - Propagation delay  — time for signal to travel the physical distance
  - Transmission delay — time to push all bits onto the wire
  - Routing hops       — each router adds microseconds

Processing latency:
  - CPU time to process the request
  - DB query execution time
  - Serialisation / deserialisation

Queuing latency:
  - Time spent waiting in a queue before being processed
  - Most variable component — spikes under load
```

**Percentiles — the right way to measure latency:**

Average latency is almost useless. One very slow request can mask thousands of fast ones. Use percentiles:

```
p50  (median)  — 50% of requests are faster than this. Your "typical" user experience.
p95            — 95% of requests are faster than this. Most users experience this or better.
p99            — 99% of requests are faster than this. Your worst 1% — often your paying customers.
p999           — 99.9% of requests are faster than this. Tail latency.
```

**Example — ShopSphere order placement latency:**
```
p50  =  120ms   ← typical user clicks "Place Order", response in 120ms
p95  =  340ms   ← 95% of users see response within 340ms
p99  =  950ms   ← 1 in 100 users waits close to a second
p999 = 4200ms   ← 1 in 1000 users waits over 4 seconds
```

The p99 and p999 are your **tail latency** — often caused by GC pauses, DB lock contention, cold cache misses, or noisy neighbours in cloud environments. At scale, tail latency affects a lot of real users — at 100,000 requests/minute your p99 is 1000 users/minute having a bad experience.

**Latency numbers every engineer should know:**

```
Operation                          Approximate Latency
─────────────────────────────────────────────────────
L1 cache reference                       0.5 ns
L2 cache reference                       7   ns
RAM reference                          100   ns
SSD random read                        150   µs
Network round trip (same datacenter)     0.5 ms
Redis GET                                1   ms
PostgreSQL query (indexed)               2   ms
Network round trip (cross-region)       30   ms
HDD seek                               10   ms
Network round trip (cross-continent)   150   ms
```

These numbers matter for system design — they tell you where your time is actually going and where caching or co-location will have the most impact.

---

## Internals — Understanding Throughput

Throughput is about **capacity** — how much work the system can sustain over time.

**What limits throughput:**
```
CPU bound    — complex computation, encryption, serialisation
Memory bound — large working sets, GC pressure
I/O bound    — disk reads/writes, DB queries
Network bound— bandwidth saturation, connection limits
```

**Little's Law — the most important throughput formula:**
```
L = λ × W

L = average number of requests in the system (queue + being processed)
λ = average arrival rate (requests per second)
W = average time each request spends in the system (latency)

Rearranged:
λ = L / W
```

**Practical example:**
- Your Order Service can hold 100 concurrent requests (`L = 100`)
- Each request takes on average 200ms to process (`W = 0.2s`)
- Maximum throughput = 100 / 0.2 = **500 requests/second**

To increase throughput you can either:
- Increase `L` — add more capacity (horizontal scaling, more threads)
- Decrease `W` — reduce latency per request (faster DB queries, caching, async processing)

---

## The Latency-Throughput Trade-off

This is where it gets interesting — and where most interview answers fall short.

**Batching — classic throughput vs latency trade-off:**
```
Option A — process each request immediately:
  Latency:    5ms per request
  Throughput: 200 requests/second

Option B — batch 100 requests, process together every 50ms:
  Latency:    up to 50ms per request (worse!)
  Throughput: 10,000 requests/second (much better!)
```

Kafka producers do this — `linger.ms` config holds messages for a few milliseconds to batch them, increasing throughput at the cost of slightly higher latency.

**Connection pooling:**
```
Without pooling:
  Each request opens new DB connection (5ms overhead) + query (2ms) = 7ms latency
  High latency, moderate throughput

With pooling (reuse existing connections):
  Each request uses existing connection (0ms overhead) + query (2ms) = 2ms latency
  Lower latency AND higher throughput — rare win-win
```

**Async processing — decouple latency from throughput:**
```
Synchronous order processing:
  Client waits for: validate → charge payment → update inventory → send email
  Latency: 800ms    Throughput: limited by slowest step

Async order processing:
  Client waits for: validate → charge payment (200ms)
  Background: update inventory + send email via queue
  Latency: 200ms (much better!)    Throughput: much higher (queue absorbs spikes)
```

This is exactly why ShopSphere uses RabbitMQ for notifications — decoupling the slow email send from the critical payment path reduces latency for the user and increases overall throughput.

---

## Bandwidth — the Related Concept

**Bandwidth** is the maximum capacity of the pipe — how much data can flow per second.

```
Bandwidth  = the width of the highway (how many lanes)
Throughput = actual cars passing per hour (never exceeds bandwidth, often less)
Latency    = time for one car to complete the journey
```

- High bandwidth does not mean low latency — a fat pipe to a distant server is still slow
- Throughput can never exceed bandwidth — it is the ceiling
- In practice throughput is always lower than bandwidth due to protocol overhead, congestion, retransmissions

**Bandwidth matters in ShopSphere for:**
- Video/image uploads to S3 — large payload, bandwidth bound
- Elasticsearch bulk indexing — high data volume
- DB replication lag — if replication bandwidth is saturated, replicas fall behind

---

## Real-World Example — ShopSphere Performance Analysis

**Scenario: ShopSphere Search Service is slow under load**

You observe:
```
Normal load  (100 req/s):   p50=80ms,  p99=200ms   ← acceptable
High load    (500 req/s):   p50=300ms, p99=2000ms  ← degraded
Extreme load (800 req/s):   p50=900ms, p99=8000ms  ← broken
```

**Diagnosis using latency + throughput thinking:**

The system handles 100 req/s fine. At 500 req/s latency explodes. This is a **queuing latency** problem — requests are piling up faster than they are processed. This is the **knee of the curve** — the point where a system transitions from healthy to overloaded.

```
Throughput ceiling of Search Service ≈ 300 req/s

Above that:
  Queue depth grows
  Requests wait longer
  Latency skyrockets
  Eventually timeouts cascade
```

**Solutions:**
- **Reduce W (per-request latency)** — cache frequent Elasticsearch queries in Redis, reducing query time from 80ms to 5ms → throughput ceiling jumps to ~2000 req/s
- **Increase L (capacity)** — horizontal scale Search Service from 2 to 6 instances → throughput ceiling triples
- **Shed load** — rate limit at API Gateway so extreme traffic doesn't collapse the system

---

## Interview Q&A

**Q: What is the difference between latency and throughput?**
Latency is the time a single operation takes from start to finish — how long one request takes. Throughput is how many operations the system can handle per unit of time — how many requests per second. A system can have high throughput but also high latency, like a batch processing pipeline that handles millions of records per hour but each record takes minutes to complete.

**Q: Why is average latency a bad metric?**
Average latency is skewed by outliers and hides the tail. If 999 requests complete in 10ms and one takes 10 seconds, the average is ~20ms which looks fine — but one in a thousand users is waiting 10 seconds. Percentiles tell the real story — p99 and p999 reveal tail latency that actually affects real users at scale.

**Q: What is tail latency and what causes it?**
Tail latency refers to p99, p999 — the slowest requests in the distribution. Common causes include JVM garbage collection pauses, database lock contention, cold cache misses, noisy neighbours in cloud environments, and thread pool exhaustion causing queuing. At scale tail latency matters a lot — at one million requests per minute, your p999 is still 1000 users per minute having a very bad experience.

**Q: How does batching affect latency and throughput?**
Batching increases throughput by amortising fixed overhead across many operations — fewer round trips, better resource utilisation. But it increases latency because individual items must wait until the batch is full or the batch window expires before being processed. Kafka's `linger.ms` is a classic example — hold messages for a few milliseconds to batch them, trading slightly higher per-message latency for dramatically higher throughput.

**Q: How would you improve throughput in ShopSphere's Order Service without adding servers?**
Reduce per-request latency — which increases throughput per Little's Law. Concretely: cache frequently read product and inventory data in Redis to eliminate DB calls, make non-critical work asynchronous via RabbitMQ (email, analytics, audit logs) to shorten the critical path, optimise DB queries with proper indexes, and increase connection pool size to reduce connection wait time. Each of these reduces W in L = λ × W, directly increasing maximum λ.

---

Say **"next"** when ready for Topic 10 — Bandwidth.
