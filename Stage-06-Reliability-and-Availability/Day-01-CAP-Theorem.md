

# Stage 6 — Reliability & Availability
## Topic 1 : CAP Theorem

---

### What is CAP Theorem?

CAP Theorem, introduced by **Eric Brewer in 2000** and formally proven by Gilbert & Lynch in 2002, states:

> **In a distributed system, you can guarantee at most 2 out of these 3 properties simultaneously.**

| Property | What it means |
|---|---|
| **C** — Consistency | Every read receives the **most recent write** or an error |
| **A** — Availability | Every request receives a **response** (not necessarily the latest data) |
| **P** — Partition Tolerance | The system **continues to operate** despite network partitions (message loss between nodes) |

---

### Why can't you have all 3?

First — understand what a **network partition** is:

```
  Node A ──────────── Node B
           ██████
         (network cut)

Node A and Node B can no longer communicate.
This WILL happen in real distributed systems.
```

A partition means two nodes can't sync. Now you're forced to choose:

---

#### Scenario: Write comes in during a partition

```
Client ──► Node A (Primary)       Node B (Replica)
              │                        │
              │  ← network dead →      │
              │                        │
          Write: price = ₹500      Still has: price = ₹400
```

**Option 1 — Choose Consistency (CP):**
- Refuse the read on Node B (return error / block)
- Data is never stale
- But Node B is **unavailable** during the partition
- ✅ Consistent ✅ Partition Tolerant ❌ Available

**Option 2 — Choose Availability (AP):**
- Node B serves the request with ₹400 (stale data)
- System stays up, no errors
- But data returned is **wrong / outdated**
- ✅ Available ✅ Partition Tolerant ❌ Consistent

**Option 3 — Choose CA (Consistency + Availability):**
- Only possible if partitions **never happen**
- This means a **single machine** / monolith
- The moment you go distributed — partitions *will* happen
- **CA systems don't exist in practice for distributed systems**

---

### The Real World Framing

> Partition Tolerance is **not optional** in distributed systems. Networks *will* fail. So the real choice is always: **CP vs AP**.

```
                  CAP Triangle

                  Consistency
                      /\
                     /  \
                    / CP \
                   /------\
                  /   CA   \
                 / (theory) \
    Availability ─────────── Partition
                    AP       Tolerance
```

---

### CP Systems — Consistency over Availability

When a partition happens → **reject requests** rather than serve stale data.

**Real examples:**
- **HBase** — refuses reads/writes during partition
- **Zookeeper** — leader election; won't serve if quorum lost
- **Redis Cluster** (in certain configs)
- **Traditional RDBMS** (PostgreSQL with synchronous replication)
- **Google Spanner** (uses TrueTime to be near-CA but fundamentally CP)

**Use when:**
- Financial transactions (bank balance, payments)
- Inventory systems (you can't oversell)
- Booking systems (seat/hotel reservation)

---

### AP Systems — Availability over Consistency

When a partition happens → **serve possibly stale data**, sync later.

**Real examples:**
- **Cassandra** — tunable consistency, defaults AP
- **DynamoDB** — eventually consistent reads by default
- **CouchDB**
- **DNS** — your DNS record update takes minutes to propagate globally; DNS servers still respond

**Use when:**
- Product catalog (showing ₹499 vs ₹500 for 2 seconds is fine)
- Social media likes/views count
- Shopping cart (eventual sync is acceptable)
- Search indexes

---

### "Eventual Consistency" — the AP bargain

AP systems say: *"I'll give you an answer now, and I promise all nodes will **eventually** agree."*

```
t=0   Write: price=₹500 → Node A
t=1   Read from Node B  → returns ₹400  (stale, but available)
t=2   Replication sync  → Node B gets ₹500
t=3   Read from Node B  → returns ₹500  ✅ (now consistent)
```

The **window of inconsistency** is called **replication lag**. In healthy systems it's milliseconds. Under load, it can be seconds.

---

### PACELC — The Modern Extension

CAP only talks about **partition scenarios**. PACELC extends it:

> Even when there's **no partition (E = Else)**, there's a tradeoff between **Latency (L)** and **Consistency (C)**.

```
If Partition:    Choose  A  or  C
Else (normal):  Choose  L  or  C
```

- **DynamoDB** → PA/EL (Available during partition, Low latency normally)
- **PostgreSQL** → PC/EC (Consistent during partition, Consistent normally — higher latency)
- **Cassandra** → PA/EL

This is why **Cassandra is so fast** — it doesn't wait for all replicas to agree on every write.

---

### ShopSphere Mapping 🛒

| Service | Choice | Why |
|---|---|---|
| **Order Service** | **CP** | Can't confirm order on stale inventory data |
| **Payment Service** | **CP** | Absolutely cannot show wrong balance |
| **Product Catalog** | **AP** | Showing ₹499 vs ₹500 briefly is fine |
| **Review Service** | **AP** | Stale review count acceptable |
| **Search Service** (ES) | **AP** | Search index can lag by seconds |
| **Cart Service** | **AP** | Eventual sync is fine, merge on conflict |
| **User Auth** (Redis session) | **CP** | Invalid session must never appear valid |

---

### Interview Questions 🎯

**Q1. Can a system be CA in distributed environment?**
> No. In a distributed system, network partitions are inevitable. When a partition occurs, you *must* choose between C and A. CA only exists theoretically for single-node systems.

**Q2. Does "Consistency" in CAP mean the same as in ACID?**
> No. CAP Consistency = all nodes see the same data at the same time (linearizability). ACID Consistency = data transitions between valid states per business rules. Different concepts, same word — classic CS naming confusion.

**Q3. Cassandra is AP — how do you make it more consistent?**
> Cassandra has tunable consistency via `QUORUM` reads/writes. Write to majority + Read from majority = strong consistency. But now you're closer to CP behavior and latency goes up.

**Q4. If you had to design ShopSphere's Order service — CP or AP?**
> CP. An order confirmation touching inventory and payment must be consistent. Using a distributed transaction (Saga pattern) across services, each step is atomic. We'd rather fail fast than confirm an order we can't fulfill.

**Q5. What's the downside of CP during a partition?**
> The system becomes unavailable. For example if a DB replica can't reach the primary, the replica stops serving reads. Users see errors or timeouts. This is the deliberate tradeoff — correctness over uptime.

---

### One-Line Summary

> **CAP says pick 2. In reality, P is mandatory, so pick C or A. Pick C when wrong data = disaster (payments, orders). Pick A when stale data is tolerable (catalog, feeds, search).**

---

Ready for **Topic 2: Availability Numbers** (99.9% vs 99.99% vs 99.999% — what downtime actually means per year)?

Type **next** 🚀
