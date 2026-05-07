# Stage 4 — Databases Deep Dive
## Topic 3: RDBMS Horizontal Scaling — Why It's Hard, Read Replicas, Connection Pooling, Partitioning

---

## 1. Why Horizontal Scaling is Hard for RDBMS

Scaling horizontally means **adding more machines** to share the load. For stateless services (your Spring Boot apps), this is trivial — spin up 10 instances, put a load balancer in front, done.

For a relational database, it's fundamentally hard. Here's why.

### The Core Problem: Shared Mutable State

Every node in a distributed system that accepts writes must agree on the current state of data. RDBMS guarantees ACID — specifically:

- **Atomicity** across a transaction that touches multiple rows
- **Consistency** enforced by constraints (FK, UNIQUE)
- **Isolation** — concurrent transactions must not interfere
- **Durability** — committed = on disk

The moment you split data across two machines, you must coordinate all four of these properties **across a network**. Networks are unreliable. Coordination is expensive.

### The Specific Killers

**Distributed transactions are expensive:**
A transaction that touches data on Node A and Node B requires a **2-Phase Commit (2PC)**:
1. Phase 1: Coordinator asks both nodes "can you commit?" → both lock resources and say yes
2. Phase 2: Coordinator says "commit" → both apply and release locks

If the coordinator crashes after Phase 1, both nodes are stuck holding locks indefinitely (**blocking problem**). 2PC also doubles network round trips and kills throughput.

**UNIQUE constraints break:**
`UNIQUE(email)` across a users table requires checking all nodes before inserting. You'd need a distributed lock or a central coordinator — both are latency nightmares.

**Foreign key constraints become cross-node:**
If `orders` is on Node A and `users` is on Node B, enforcing `orders.user_id FK → users.id` requires a network call on every insert.

**Joins across nodes:**
A JOIN between tables on different nodes means fetching data over the network and joining in memory. Often slower than not splitting at all.

This is why **NoSQL databases scale horizontally easily** — they drop constraints, relax consistency, and eliminate cross-row transactions.

---

## 2. Read Replicas

The pragmatic first answer to scaling a relational DB: **don't split writes, but distribute reads**.

### How It Works

```
Application
    │
    ├──── Writes ────→ Primary (Master)
    │                       │
    │                  WAL streaming
    │                  ↓         ↓
    ├──── Reads ─────→ Replica 1  Replica 2
    └──── Reads ─────→ Replica 3
```

PostgreSQL ships WAL records from the primary to replicas in real time. Replicas apply these WAL records to stay in sync.

**Synchronous replication:** Primary waits for at least one replica to confirm WAL receipt before returning COMMIT. Zero data loss, but adds latency to every write.

**Asynchronous replication (default):** Primary commits immediately, ships WAL in the background. Faster writes, but replica may lag by milliseconds to seconds.

### What You Route Where

| Query Type | Target |
|---|---|
| Order placement, payment, user signup | Primary |
| Product catalog browsing | Replica |
| Order history listing | Replica (acceptable slight lag) |
| Admin dashboard analytics | Dedicated replica |
| Inventory check during checkout | Primary (must be current) |

### The Replication Lag Problem

This is the **eventual consistency trade-off** in RDBMS replicas. A user places an order, then immediately refreshes "My Orders" — the read might hit a lagging replica and show nothing. User thinks the order failed.

**Solutions:**
- **Read-your-writes consistency:** After a write, route that user's reads to primary for a short window (sticky session or header-based routing)
- **Session tokens:** Write a version token on commit, pass it to the replica — replica only serves the read if it has caught up to that version (PostgreSQL 14+ supports this with LSN tracking)
- **Accept it for non-critical reads:** Product views, search results, dashboards — slight lag is fine

---

## 3. Connection Pooling

### Why Creating a New Connection Per Request Kills You

A PostgreSQL connection is **not cheap**:

- PostgreSQL forks a new backend **process** per connection (not a thread — a full OS process)
- Each process uses ~5–10MB of memory
- TCP handshake + SSL negotiation + auth = ~30–50ms of setup overhead

A Spring Boot service handling 500 req/s with a new connection per request = 500 new connections/second = 500 new OS processes/second = system melts.

### What a Connection Pool Does

A pool pre-creates N connections at startup and keeps them alive. Requests borrow a connection, use it, return it.

```
Spring Boot App (100 threads)
        │
        ▼
  ┌─────────────────────────┐
  │   HikariCP Pool (20)    │  ← 20 real PG connections kept alive
  │  [conn][conn][conn]...  │
  └─────────────────────────┘
        │
        ▼
   PostgreSQL Primary
```

100 threads share 20 connections. A thread that needs a connection waits in queue (microseconds) rather than paying the 50ms connection setup cost.

### HikariCP — What You're Already Using

Spring Boot auto-configures HikariCP. Key settings:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # max connections to DB
      minimum-idle: 5              # keep at least 5 alive
      connection-timeout: 30000    # wait up to 30s for a connection
      idle-timeout: 600000         # close idle connections after 10m
      max-lifetime: 1800000        # recycle connections every 30m
```

**How to size the pool:**

A common formula: `pool_size = (core_count * 2) + effective_spindle_count`

For a 4-core app server targeting a 4-core DB server: `(4 * 2) + 1 = 9`. HikariCP's own research shows **small pools outperform large pools** because large pools cause DB-side process contention. Counter-intuitive but true.

**What kills you:**
- Pool too small → threads queue up, latency spikes, timeouts
- Pool too large → DB gets overwhelmed by concurrent processes, memory exhaustion
- `max-lifetime` not set → stale connections that DB has killed silently cause errors on next use

### PgBouncer — Pool at the Infrastructure Level

HikariCP pools connections **per application instance**. If you have 20 ShopSphere instances each with a pool of 20, that's 400 connections hitting PostgreSQL — still too many.

**PgBouncer** sits between apps and PostgreSQL as a dedicated connection proxy:

```
20 ShopSphere Instances (pool=20 each)  →  PgBouncer  →  PostgreSQL (10 connections)
      400 app-side connections                              10 real connections
```

PgBouncer modes:
- **Session mode:** One server connection per client session. No benefit over app-level pooling.
- **Transaction mode:** Server connection held only for the duration of a transaction, then returned. Most efficient — supports far more app connections than DB connections.
- **Statement mode:** Connection returned after every single statement. Breaks multi-statement transactions — rarely used.

ShopSphere production setup: HikariCP on each service (reduces individual service overhead) + PgBouncer in transaction mode (caps total DB connections).

---

## 4. Partitioning

When a single table gets so large that even with indexes it's slow — you **partition** it. Partitioning splits one logical table into multiple physical storage units, still on the **same machine** (unlike sharding, which splits across machines).

### Range Partitioning

Split by a range of values — almost always by date.

```sql
CREATE TABLE orders (
    order_id UUID,
    user_id  UUID,
    created_at TIMESTAMPTZ,
    ...
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

Query: `WHERE created_at >= '2024-06-01'` → PostgreSQL does **partition pruning** — only scans `orders_2024`, ignores `orders_2023`. Massively faster.

### Hash Partitioning

Split by hash of a column. Used when no natural range exists and you want even distribution.

```sql
PARTITION BY HASH (user_id);
-- Partition 0: user_ids where hash(user_id) % 4 = 0
-- Partition 1: hash(user_id) % 4 = 1
-- ...
```

### List Partitioning

Split by discrete values — e.g., region.

```sql
PARTITION BY LIST (region);
-- Partition APAC: ('IN', 'SG', 'AU')
-- Partition US:   ('US', 'CA')
```

### What Partitioning Gives You

| Benefit | How |
|---|---|
| Faster queries | Partition pruning eliminates irrelevant partitions |
| Faster bulk deletes | `DROP TABLE orders_2021` is instant vs `DELETE FROM orders WHERE year=2021` (which locks and scans) |
| Index size reduction | Each partition has its own smaller index — fits in memory better |
| Parallel query | PG can scan multiple partitions in parallel |

### What Partitioning Does NOT Give You

- **It doesn't scale writes across machines** — all partitions are on the same server
- **It doesn't eliminate hot spots** — if everyone queries the current month, `orders_2024` is still the hot partition
- **Cross-partition queries pay full price** — `SELECT COUNT(*) FROM orders` scans all partitions

---

## 5. ShopSphere Mapping

**Orders table — range partition by `created_at`:**
Orders from 2022 are rarely queried. Partition by year → old data stays on cold storage partitions, 2024 partition is small and hot.

**Connection pooling setup:**
```
order-service (3 instances × pool=10) 
product-service (5 instances × pool=10)
         ↓
     PgBouncer (transaction mode, max_server_conns=30)
         ↓
  PostgreSQL Primary (max_connections=100, comfortable)
```

**Read replica routing in Spring Boot:**
```java
@Transactional(readOnly = true)  // Spring routes to replica DataSource
public List<Order> getOrderHistory(UUID userId) { ... }

@Transactional  // Routes to primary
public Order placeOrder(OrderRequest req) { ... }
```

Use `AbstractRoutingDataSource` to implement primary/replica routing based on transaction read-only flag.

---

## 6. Interview Q&A

**Q: Why can't you just add more PostgreSQL nodes and have them all accept writes?**
A: Because you'd need distributed transactions (2PC) to maintain ACID across nodes — expensive, slow, and have failure modes like the blocking problem. You'd also need distributed enforcement of UNIQUE and FK constraints. The coordination overhead often exceeds the benefit. Multi-master PG exists (like Citus) but with significant trade-offs.

**Q: What's replication lag and how do you handle it for critical reads?**
A: Replication lag is the delay between a write being committed on the primary and being visible on a replica. For critical reads (e.g., inventory during checkout), always read from primary. For read-your-writes consistency after user actions, track the WAL LSN at commit time and only serve reads from a replica that has reached that LSN.

**Q: How do you decide connection pool size?**
A: Start with `(2 × cores) + disk_spindles` per app instance. Then measure: if threads are queuing for connections, pool is too small. If DB shows high process contention or memory pressure, pool is too large. PgBouncer in transaction mode is the real solution for high instance counts — it decouples app-side concurrency from DB-side connections.

**Q: What's the difference between partitioning and sharding?**
A: Partitioning splits a table into physical segments on the **same machine** — it's a storage and query optimization. Sharding splits data across **multiple machines** — it's a horizontal scaling strategy. Partitioning is transparent to the application. Sharding usually requires application-level awareness of which shard to target.

**Q: When does partition pruning NOT work?**
A: When the query doesn't filter on the partition key. `SELECT * FROM orders WHERE user_id = 42` on a table partitioned by `created_at` — PostgreSQL must scan all partitions because it doesn't know which partition holds that user's orders. Always filter on the partition key for pruning to kick in.

---

Ready for **Topic 4: Database Read Replicas — offloading read traffic and the eventual consistency trade-off** (deep dive)?
