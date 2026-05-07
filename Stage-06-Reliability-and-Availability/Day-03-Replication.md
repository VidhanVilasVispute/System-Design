# Stage 6 — Reliability & Availability
## Topic 3 : Replication

---

### What is Replication?

> **Replication = keeping copies of your data on multiple nodes so that if one dies, others can serve.**

It solves three problems simultaneously:

```
Problem 1: Single Point of Failure
  Primary dies → replica takes over → no data loss

Problem 2: Read Scalability
  10,000 reads/sec → spread across 5 replicas → 2,000 each

Problem 3: Geographic Latency
  User in Mumbai → read from Mumbai replica (not US)
  User in US     → read from US replica
```

---

### The Two Fundamental Models

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   SYNCHRONOUS              ASYNCHRONOUS             │
│                                                     │
│   Write → Primary          Write → Primary          │
│         → wait for               → ack immediately  │
│           replica ack            → replicate later  │
│         → ack client                                │
│                                                     │
│   Stronger consistency     Faster writes            │
│   Slower writes            Risk of data loss        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

### Synchronous Replication — Deep Dive

```
Client
  │
  │  1. Write Request (INSERT order)
  ▼
Primary DB
  │
  │  2. Write to own WAL + disk
  │
  │  3. Forward to Replica
  ▼
Replica DB
  │
  │  4. Write confirmed
  │
  │  5. Send ACK back to Primary
  ▼
Primary DB
  │
  │  6. Now ACK the client ✅
  ▼
Client ← "Write successful"
```

**WAL = Write-Ahead Log** — the backbone of replication (PostgreSQL, MySQL both use this)

```
WAL Entry format:
  LSN (Log Sequence Number) | Operation | Table | Data
  00000001/3A4B0000         | INSERT    | orders| {id:1, total:999}
  00000001/3A4B0100         | UPDATE    | inventory | {qty: 49}
```

Every change is first written to WAL, then sent to replica. Replica replays the WAL to stay in sync.

#### Synchronous — What You Get

```
✅ Zero data loss (RPO = 0)
   Primary and Replica are always identical

✅ Failover is instant and clean
   Replica can become primary with no data gap

❌ Write latency increases
   Latency = Primary write time + Network RTT + Replica write time
   If replica is in another AZ → +2–5ms per write

❌ Replica slowness = Primary slowness
   Replica under load? Primary waits. Primary stalls. Clients timeout.

❌ If replica is unreachable:
   Option A: Block writes  → availability drops (CP behavior)
   Option B: Skip replica  → you're now effectively async
```

**PostgreSQL's exact config:**
```sql
-- postgresql.conf
synchronous_commit = on           -- default, synchronous
synchronous_standby_names = '*'   -- wait for ALL standbys

-- Per transaction override
SET synchronous_commit = off;     -- make this one async
```

---

### Asynchronous Replication — Deep Dive

```
Client
  │
  │  1. Write Request
  ▼
Primary DB
  │
  │  2. Write to own WAL + disk
  │
  │  3. ACK client immediately ✅  ← no waiting
  │
  │  4. (background) stream WAL to replica
  ▼
Replica DB  ← catches up eventually
```

#### The Replication Lag Problem

```
t=0ms   Write: inventory qty = 49  → Primary
t=0ms   Client gets ACK ✅
t=50ms  Replica still shows qty = 50  ← STALE
t=80ms  WAL entry arrives at Replica
t=82ms  Replica applies it → qty = 49  ✅

During t=0 to t=82ms:
  → Read from Replica = WRONG DATA
  → This window is "replication lag"
```

**Real cost of replication lag:**

```
Scenario: Flash sale on ShopSphere

t=0   Stock = 1 unit
t=1   User A buys → Primary: stock = 0
t=2   User B reads replica → sees stock = 1 ← lag!
t=3   User B places order  → oversell! 💥

Fix: Route inventory reads to Primary during checkout
     (read-your-writes consistency)
```

#### Asynchronous — What You Get

```
✅ Write latency = Primary only
   No waiting for replica → fast writes

✅ Replica failure doesn't affect Primary
   Primary keeps accepting writes independently

✅ Replicas can be far away (cross-region)
   Async tolerates high network latency

❌ Data loss on Primary crash
   WAL entries not yet sent to replica → GONE
   RPO > 0 (seconds to minutes of data loss possible)

❌ Replication lag during high write load
   Replica falls behind → reads increasingly stale

❌ Failover is messy
   Replica may be behind → need to decide:
   promote anyway (lose data) or wait (stay down)
```

---

### Semi-Synchronous — The Pragmatic Middle Ground

Used by **MySQL Group Replication, Galera Cluster.**

```
Primary writes → waits for ACK from AT LEAST ONE replica
                 (not all replicas, just one)

If that one replica dies → falls back to async automatically

Balance:
  ✅ At least one copy always confirmed
  ✅ Better than pure async for RPO
  ✅ Better than pure sync for latency
  ❌ Still possible to lose data if Primary + that one replica both die
```

---

### Replication Topologies

#### Single Leader (Master-Replica) — Most Common

```
        ┌─────────────┐
        │   Primary   │◄── All Writes
        └──────┬──────┘
               │ WAL stream
       ┌───────┼───────┐
       ▼       ▼       ▼
   Replica1 Replica2 Replica3
      │
   All Reads (can be spread)
```

- Simple, easy to reason about
- PostgreSQL, MySQL default setup
- **ShopSphere uses this**

---

#### Multi-Leader (Multi-Master)

```
  Region: Mumbai          Region: Singapore
  ┌─────────────┐         ┌─────────────┐
  │  Primary A  │◄──────►│  Primary B  │
  │  (writes)   │  sync   │  (writes)  │
  └─────────────┘         └─────────────┘

  Both accept writes. Sync with each other.
```

- Used for geo-distributed writes
- **Conflict resolution is HARD**
- Example: User updates profile in Mumbai and Singapore simultaneously → who wins?
- CouchDB, Cassandra use this

---

#### Leaderless (Dynamo-style)

```
Client writes to ANY node (no designated primary)

  Write to 3 of 5 nodes (W=3)
  Read  from 3 of 5 nodes (R=3)
  W + R > N → guaranteed to see latest write

        Node1  Node2  Node3  Node4  Node5
Write:   ✅     ✅     ✅     ✗      ✗
Read:    ✅     ✅           ✅
         └──── 2 of 3 agree: latest value ───┘
```

- Cassandra, DynamoDB, Riak
- No single point of failure
- Tunable consistency via quorum

---

### Read Replicas in Practice

```
Application Layer
       │
       ├── Writes ──────────────► Primary DB
       │                              │
       └── Reads ──► Read Replica 1   │ (async replication)
                  └► Read Replica 2 ◄─┘

Config in Spring Boot / ShopSphere:

@Primary
@Bean
DataSource writeDataSource() { return primaryDS; }

@Bean
DataSource readDataSource() { return replicaDS; }

@Transactional(readOnly = true)  // routes to replica
public List<Product> getProducts() { ... }

@Transactional  // routes to primary
public Order placeOrder() { ... }
```

---

### ShopSphere Replication Strategy 🛒

```
┌──────────────────────────────────────────────────────┐
│                  ShopSphere DB Layer                  │
│                                                       │
│  Order / Payment Service                              │
│  ┌─────────────┐  synchronous   ┌─────────────┐      │
│  │  Primary    │──────────────► │  Replica    │      │
│  │  (AZ-1)     │                │  (AZ-2)     │      │
│  └─────────────┘                └─────────────┘      │
│  RPO = 0, RTO < 30s  ← money, can't lose data        │
│                                                       │
│  Product / Review / Notification Service              │
│  ┌─────────────┐  asynchronous  ┌─────────────┐      │
│  │  Primary    │──────────────► │  Replica    │      │
│  │  (AZ-1)     │                │  (AZ-2)     │      │
│  └─────────────┘                └─────────────┘      │
│  RPO = seconds, faster writes ← stale reads = fine   │
│                                                       │
│  Search Service (Elasticsearch)                       │
│  ┌────────────────────────────────────┐               │
│  │  3-node cluster, 1 primary shard  │               │
│  │  + 1 replica shard per node       │               │
│  │  Async by nature                   │               │
│  └────────────────────────────────────┘               │
└──────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What is WAL and how does replication use it?**
> WAL (Write-Ahead Log) records every change before applying it to data files. For replication, the Primary streams WAL entries to replicas. Replicas replay this log to stay in sync. It's also used for crash recovery — if Primary dies mid-write, WAL lets it recover on restart.

**Q2. You have async replication. Primary crashes. What happens?**
> Replica is promoted to Primary. But any WAL entries not yet sent to the replica are lost — that's your RPO window. You can minimize this with semi-sync or by tuning `wal_keep_size` in PostgreSQL to buffer more WAL, giving replica more time to catch up before entries are recycled.

**Q3. How do you handle read-your-own-writes consistency with replicas?**
> After a write, route subsequent reads for that user to Primary for a short window (e.g., 1–2 seconds), or use a sticky session to always route that user's reads to Primary. Some systems use a "read-after-write" token — client sends the write's LSN, replica refuses to serve until it reaches that LSN.

**Q4. Sync replication — replica goes down. What does Primary do?**
> Two choices: block writes until replica recovers (maintains durability, kills availability) or demote to async mode temporarily (maintains availability, risks data loss). PostgreSQL's `synchronous_commit = remote_apply` with `ANY 1` quorum gives you a middle ground — write succeeds if any one standby confirms.

**Q5. What's the difference between replication and sharding?**
> Replication = **same data** on multiple nodes (for redundancy + read scaling). Sharding = **different data** on different nodes (for write scaling + storage scaling). They're complementary — you shard AND replicate each shard.

---

### One-Line Summary

> **Synchronous replication = zero data loss, slower writes, replica health affects primary. Asynchronous = fast writes, possible data loss on crash, replication lag creates stale reads. Pick sync for money, pick async for scale.**

---

Ready for **Topic 4: Failover** — automatic replica promotion, how failover detection works, split-brain problem, and what happens in that dangerous gap between primary dying and replica taking over?

Type **next** 🚀
