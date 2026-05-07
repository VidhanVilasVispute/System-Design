
# Stage 7 — Distributed Systems
## Topic 1: Why Distributed Systems Are Hard

---

## The Core Problem

In a single-process app, everything is simple:

```
Function A calls Function B
→ Either it works, or it throws an exception
→ You always know which one happened
```

In a distributed system:

```
Service A calls Service B over the network
→ It might work ✅
→ It might fail ❌
→ It might succeed but the response got lost 🤷
→ It might be running slow and not answered yet ⏳
→ You genuinely CANNOT TELL which one happened
```

That last point is the entire reason distributed systems are hard. It's not performance. It's not scale. It's **uncertainty**.

---

## The Three Fundamental Problems

### 1. 🔴 Partial Failures

In a single machine — if a process crashes, **everything** crashes together. State is consistent (even if lost).

In a distributed system — **one node can fail while others keep running**.

```
ShopSphere Example:
───────────────────────────────────────────────────────
User places Order
  → order-service saves to DB ✅
  → calls payment-service      ❌ payment-service is down
  → calls inventory-service    ✅ (never got called)

Result: Order exists in DB, payment was never charged,
        inventory was never decremented.

Who's responsible for cleaning this up? There's no automatic rollback.
That's a PARTIAL FAILURE.
───────────────────────────────────────────────────────
```

This is why patterns like **Saga**, **2PC**, and the **Outbox Pattern** exist — we'll cover all of them this stage.

---

### 2. 🔴 Network Unreliability

The network between your services is **not a reliable pipe**. It:

- Drops packets
- Reorders packets
- Delays packets arbitrarily
- Duplicates packets (yes — your request can arrive **twice**)

Peter Deutsch codified this as the **8 Fallacies of Distributed Computing**. The #1 fallacy engineers believe:

> ❌ "The network is reliable"

What actually happens:

```
service-a ──[request]──────────────────────────────▶ service-b
                                                         ✅ processed

service-a ◀──────────────────────── [response lost] ────
           ⏳ timeout!

service-a thinks: "Did service-b get it? I don't know."
service-a thinks: "Should I retry? What if it processed twice?"
```

This creates the concept of **idempotency** — your operations must be safe to call multiple times with the same result. That's why Stripe payment APIs take an `idempotency-key`.

---

### 3. 🔴 No Global Clock

On a single machine, you can call `System.currentTimeMillis()` and get an authoritative "now".

Across machines — **every node has its own clock**, and they drift.

```
Node A clock: 10:00:00.000
Node B clock: 10:00:00.047   ← 47ms ahead

Event on A at "10:00:00.100"
Event on B at "10:00:00.090"

Wall clock says: B happened first
Reality: We have NO IDEA. A's clock might be the accurate one.
```

**Why does this matter?**

```
ShopSphere flash sale:
──────────────────────────────────────────────────────
User1 buys last item at Node A → timestamp 10:00:00.100
User2 buys last item at Node B → timestamp 10:00:00.090

B's timestamp is "earlier" → does User2 win?
But maybe A's clock is correct and A was actually first!

No global clock = no reliable "happened before" ordering.
```

This is exactly why **Lamport Clocks** and **Vector Clocks** were invented (Topics 3 & 4).

---

## The CAP Theorem — The Unavoidable Tradeoff

Every distributed system must choose **2 of 3**:

```
         Consistency
              ▲
              │
              │
    CP ───────┼─────── CA
              │
              │
    ──────────┼──────────▶
              │        Availability
    P ────────┘
  (Partition
   Tolerance)
```

| Property | Meaning |
|---|---|
| **C** — Consistency | Every read gets the most recent write (or an error) |
| **A** — Availability | Every request gets a response (might be stale) |
| **P** — Partition Tolerance | System keeps working even if nodes can't communicate |

**The Catch:** In any real distributed system, **network partitions WILL happen**. You cannot opt out of P. So the real choice is:

> **CP** (stay consistent, go unavailable during partition)
> vs
> **AP** (stay available, might return stale data)

```
Real systems:
─────────────────────────────────────────────────
PostgreSQL (single node)  → CA  (no partition)
ZooKeeper / etcd          → CP  (consistency over availability)
Cassandra / DynamoDB      → AP  (availability over consistency)
Redis (cluster mode)      → AP  (can serve stale reads)
─────────────────────────────────────────────────
```

ShopSphere's own choices:
- **PostgreSQL** per service → CP per service
- **Redis cache** → AP (can serve slightly stale product prices)
- **Elasticsearch** → AP (search index may lag behind writes)
- **Kafka** → CP (durable, ordered log)

---

## PACELC — The More Honest Model

CAP only talks about partition scenarios. **PACELC** extends it:

> "Even when there's **no partition (E)**, you still choose between **Latency (L)** and **Consistency (C)**"

```
PACELC = If Partition: (A vs C), Else: (L vs C)

Cassandra = PA/EL → Available during partition, Low latency else
PostgreSQL = PC/EC → Consistent during partition, Consistent else
DynamoDB   = PA/EL → configurable but defaults to available
```

This is why Cassandra is blazing fast — it's trading consistency for latency even in normal operation.

---

## The Fallacies in ShopSphere Terms

| Fallacy | ShopSphere Reality |
|---|---|
| Network is reliable | Feign calls between services can timeout/fail |
| Latency is zero | A chain of 5 service calls compounds latency |
| Bandwidth is infinite | Event payloads on Kafka need size discipline |
| Network is secure | Internal mTLS or auth headers needed (your FeignAuthInterceptor) |
| Topology doesn't change | Pods restart, IPs change — use service discovery |
| There is one admin | Multiple team members deploy services independently |
| Transport cost is zero | Serialization (JSON/Avro) has real CPU cost |
| Network is homogeneous | Mix of REST, Kafka, Redis protocols in same system |

---

## Interview Angles 🎯

**Q: Why can't we just use transactions across microservices like we do in a monolith?**
> Because distributed transactions require network calls, and any node can fail mid-transaction. 2PC solves this but blocks on failure — most systems prefer Saga with compensating transactions instead.

**Q: What does "eventual consistency" mean?**
> Given no new writes, all replicas will *eventually* converge to the same value. You trade immediate consistency for availability and lower latency. Cassandra and DynamoDB default to this.

**Q: A service timed out — should you retry?**
> Only if the operation is **idempotent**. Non-idempotent retries (like "charge payment") can cause duplicate side effects. Use idempotency keys or check-then-act patterns.

**Q: Why can't you use timestamps to order distributed events?**
> Clock skew — each machine's clock drifts independently. A later wall clock time does not mean a later event. Lamport clocks and vector clocks solve this with logical time.

---

## What's Coming Next

This topic was the **"why it's hard"** foundation. Everything else this stage is a solution:

```
Problem                     → Solution (covered next)
──────────────────────────────────────────────────────
No global clock             → Lamport Clocks, Vector Clocks
Nodes disagree on values    → Raft/Paxos, Quorum Reads/Writes
Mid-transaction crash       → 2PC, Saga Pattern
Lost events on crash        → Outbox Pattern, WAL
Which node leads?           → Leader Election
```

---

Ready for **Topic 2: Consistency Models**? (strong, eventual, causal, read-your-writes — with real system mappings) — just say **next** 🚀
