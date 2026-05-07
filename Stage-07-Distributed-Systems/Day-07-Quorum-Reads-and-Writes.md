# Stage 7 — Distributed Systems
## Topic 7: Quorum Reads/Writes

---

## The Problem We're Solving

Strong consistency is safe but slow — every write waits for all replicas.
Eventual consistency is fast but unsafe — reads might get stale data.

**What if you could dial in exactly how much consistency you need, per operation?**

That's what quorums give you. A single number that lets you trade off consistency against availability and latency — and the math always tells you whether you're safe.

---

## The Setup

Assume a key-value store with **N replicas** of every piece of data.

```
N = 3 replicas for every key

      Key "stock:shoe-42" = 10

      Replica 1: stock = 10
      Replica 2: stock = 10
      Replica 3: stock = 10
```

Two tunable parameters:

```
W = write quorum  → how many replicas must confirm a write before success
R = read quorum   → how many replicas must respond before returning a value
```

You pick W and R. The system enforces them. That's it.

---

## The Magic Formula

```
Strong consistency is guaranteed when:

        R + W > N

That's it. One inequality.
```

Why does this work?

```
If R + W > N:
  → The set of nodes you wrote to
     and the set of nodes you read from
     MUST overlap by at least one node.

  That overlapping node has the latest write.
  So your read ALWAYS sees the most recent write. ✅

If R + W ≤ N:
  → The write set and read set might not overlap at all.
  → You might read from nodes that haven't seen the write.
  → Eventual consistency — stale reads possible.
```

Visualized for N=3:

```
N=3, W=2, R=2  →  R+W = 4 > 3  ✅ STRONG CONSISTENCY

Write goes to:   [Replica1 ✅, Replica2 ✅, Replica3 ❌]
Read comes from: [Replica2 ✅, Replica3 ❌]

Overlap: Replica2 ← has the latest write
Read gets fresh data. ✅

─────────────────────────────────────────────────────────────
N=3, W=1, R=1  →  R+W = 2 ≤ 3  ❌ EVENTUAL CONSISTENCY

Write goes to:   [Replica1 ✅, Replica2 ❌, Replica3 ❌]
Read comes from: [Replica3 ❌]

No overlap. Replica3 has old value.
Read gets stale data. ⚠️
```

---

## Common Quorum Configurations

```
N=3 examples:

Config       W    R    R+W    Consistency      Latency
──────────────────────────────────────────────────────────────
ALL          3    3     6     Strongest         Slowest
QUORUM       2    2     4     Strong ✅          Balanced ← sweet spot
ONE          1    1     2     Eventual          Fastest
Write-heavy  3    1     4     Strong + fast R   Slow W
Read-heavy   1    3     4     Strong + fast W   Slow R
```

### The Sweet Spot: QUORUM

For N=3, W=2, R=2:

```
Tolerates:
  1 replica being down for writes  (only need 2 of 3)
  1 replica being down for reads   (only need 2 of 3)
  1 replica being stale            (at least 1 overlapper has latest)

This is what Cassandra calls QUORUM consistency level.
Best balance of consistency + availability for most production workloads.
```

---

## How Cassandra Implements Quorums

Cassandra is the canonical example — it exposes quorum tuning directly to the application developer.

### Cassandra Consistency Levels

```java
// Cassandra CQL — you set consistency per query

// Eventual consistency — fastest, can be stale
SELECT * FROM products WHERE id = ? 
  CONSISTENCY ONE;

// Strong consistency — guaranteed fresh
SELECT * FROM products WHERE id = ?
  CONSISTENCY QUORUM;

// Strongest — all replicas must respond
SELECT * FROM products WHERE id = ?
  CONSISTENCY ALL;
```

```
Cassandra consistency levels:
──────────────────────────────────────────────────────────────────
ONE          W/R from 1 replica   — fastest, eventual
TWO          W/R from 2 replicas
THREE        W/R from 3 replicas
QUORUM       W/R from majority    — (N/2)+1 replicas
LOCAL_QUORUM Quorum within DC only  ← most used in multi-DC setups
EACH_QUORUM  Quorum in EVERY DC
ALL          W/R from all replicas — strongest, least available
──────────────────────────────────────────────────────────────────
```

### Cassandra Replication Factor

In Cassandra you set N (replication factor) per keyspace:

```sql
CREATE KEYSPACE shopsphere
  WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,   -- 3 replicas in datacenter 1
    'dc2': 3    -- 3 replicas in datacenter 2
  };
```

For N=3, QUORUM means W=2, R=2 → R+W=4 > 3 → strong consistency. ✅

---

## Multi-Datacenter Quorums

This is where it gets really powerful:

```
ShopSphere deployed in 2 DCs: Mumbai + Singapore
N = 6 total (3 per DC)

QUORUM (global):  need 4/6 responses
  → If one DC goes down (3 nodes lost), need 4 but only have 3 ❌
  → Cluster becomes unavailable for strong reads/writes

LOCAL_QUORUM:     need 2/3 responses within SAME DC
  → If Singapore DC goes down entirely:
      Mumbai still has 3 nodes, needs 2 → still available ✅
  → Each DC can serve its local traffic independently

This is why LOCAL_QUORUM is the standard choice for
multi-DC Cassandra deployments.
```

```
                   Mumbai DC              Singapore DC
                 ┌────────────┐          ┌────────────┐
Write request ──▶│ R1  R2  R3 │          │ R4  R5  R6 │
                 │ ✅  ✅  ❌ │  async   │ (async)    │
                 └────────────┘ ───────▶ └────────────┘
                  LOCAL_QUORUM
                  2/3 confirmed ✅
                  Return success to client immediately
                  Singapore syncs asynchronously
```

---

## How DynamoDB Implements Quorums

DynamoDB abstracts quorum behind two simpler options:

```
Eventually Consistent Read (default):
  R = 1 (reads from any replica, might be stale)
  Cheapest, fastest
  Half the read capacity units

Strongly Consistent Read:
  R = majority (quorum internally, N=3 → R=2)
  Guaranteed to return latest write
  Costs twice the read capacity units

Write (always):
  W = majority (quorum, N=3 → W=2)
  DynamoDB doesn't let you tune W directly
```

```java
// DynamoDB Java SDK

// Eventually consistent (default) — cheaper
GetItemRequest eventual = GetItemRequest.builder()
    .tableName("products")
    .key(key)
    .consistentRead(false)    // ← eventual
    .build();

// Strongly consistent — for inventory, payment, order state
GetItemRequest strong = GetItemRequest.builder()
    .tableName("products")
    .key(key)
    .consistentRead(true)     // ← strong (quorum read internally)
    .build();
```

---

## Quorum Failure Scenarios

### Write Fails — What Happens?

```
N=3, W=2
Write to stock = 8

  Replica1: written  ✅  stock=8
  Replica2: written  ✅  stock=8
  Replica3: network error ❌  stock=10 (stale)

W=2 achieved → write SUCCEEDS. Client told "OK".

But Replica3 has stale data.
Next read with R=2 → reads Replica2 + Replica3
  Replica2: 8
  Replica3: 10  ← stale!

How does the system know which is right?
```

### Read Repair — The Self-Healing Mechanism

Cassandra and Dynamo solve this automatically:

```
Read Repair:
────────────────────────────────────────────────────────────
1. Coordinator sends read to R replicas simultaneously
2. Compares responses — detects mismatch

   Replica2 returns: {value: 8, timestamp: T2}
   Replica3 returns: {value: 10, timestamp: T1}

3. T2 > T1 → value=8 is newer
4. Returns value=8 to client ✅
5. ASYNCHRONOUSLY writes value=8 back to Replica3

Replica3 is now healed. ✅
────────────────────────────────────────────────────────────
```

This is called **read repair** — the act of reading is also an opportunity to fix inconsistency. Cassandra does this on every read (or configurable percentage).

### Hinted Handoff — Write Repair

What if a replica is down during a write?

```
N=3, W=2
Replica3 is DOWN

Write goes to Replica1 ✅, Replica2 ✅, Replica3 ❌

W=2 achieved → write succeeds.

But Replica3 will miss this write permanently... unless:

Hinted Handoff:
  The coordinator stores a "hint" locally:
  "When Replica3 comes back, send it this write"

  Replica3 recovers →
  Coordinator delivers the hint →
  Replica3 catches up ✅
```

Combined, Read Repair + Hinted Handoff give Cassandra **self-healing** eventual consistency — even with W=1, R=1, data converges.

---

## Sloppy Quorum — The Availability Trick

DynamoDB's original Dynamo paper introduced **sloppy quorum**:

```
Normal quorum: must get W/R responses from the "home" replicas

Sloppy quorum: if a home replica is unreachable,
               use a DIFFERENT node temporarily

Example:
  Home replicas for key K: [Node1, Node2, Node3]
  Node2 is down

  Normal quorum: wait for Node2 or fail
  Sloppy quorum: use Node4 as temporary stand-in

  Node4 accepts the write, stores it with a hint:
  "This write belongs to Node2, deliver when it's back"

  When Node2 recovers → hinted handoff → Node2 gets the write
```

**Trade-off:** Sloppy quorum improves availability during node failures, but the "quorum" no longer guarantees overlap with the home replicas. Consistency guarantee is weakened during the failure window.

Cassandra offers this via `speculative_retry` and `read_repair_chance`.

---

## Quorum and Cassandra's Coordinator Role

```
Client writes to Cassandra:
────────────────────────────────────────────────────────────
1. Client connects to ANY Cassandra node → that node is
   the COORDINATOR for this request

2. Coordinator determines which nodes own the key
   (via consistent hashing on partition key)

3. Coordinator sends write to all replica nodes

4. Waits for W acknowledgments

5. Returns success to client

6. Remaining replicas catch up asynchronously

                      ┌──────────────┐
          Client ────▶│  Coordinator │ (any node)
                      └──────┬───────┘
                             │ write to all 3 replicas
               ┌─────────────┼─────────────┐
               ▼             ▼             ▼
           Replica1      Replica2      Replica3
              ✅             ✅            ❌
               └─────────────┘
               2 ACKs = QUORUM met → success returned
                                     Replica3 catches up later
────────────────────────────────────────────────────────────
```

---

## ShopSphere — Applying Quorum Thinking

Even though ShopSphere uses PostgreSQL (not Cassandra), the quorum mindset applies everywhere:

```
Component           Quorum Thinking Applied
──────────────────────────────────────────────────────────────────────
PostgreSQL + Patroni  W=synchronous_commit, R=from primary
                      synchronous_standby_names = '1 (standby1)'
                      → Write confirmed only when 1 standby ACKs WAL
                      → Quorum write essentially

Redis Sentinel        3 Sentinels, majority (2) must agree on failover
                      That's a quorum election

Elasticsearch         primary_shard + number_of_replicas
                      write_consistency (quorum) before indexing confirms

Kafka                 acks=all + min.insync.replicas=2
                      → Producer waits for 2 ISR brokers to ACK
                      → That's a quorum write on Kafka
──────────────────────────────────────────────────────────────────────
```

### Kafka Quorum Config in ShopSphere

```yaml
# docker-compose.yml — Kafka producer config
KAFKA_CFG_MIN_INSYNC_REPLICAS: 2     # at least 2 brokers must be in-sync
```

```java
// KafkaProducerConfig.java in order-service
props.put(ProducerConfig.ACKS_CONFIG, "all");
// "all" = leader waits for ALL ISR (in-sync replicas) to ACK
// combined with min.insync.replicas=2 → quorum write ✅

// If only 1 broker is in-sync → producer gets NotEnoughReplicasException
// Better to fail loudly than silently lose an order event
```

---

## Picking the Right Quorum for ShopSphere Services

```
Service / Data             N   W   R   R+W   Rationale
──────────────────────────────────────────────────────────────────────────
Order state (critical)     3   2   2    4    Strong — never lose order
Payment status             3   3   2    5    Strictest — financial data
Inventory count            3   2   2    4    Strong — prevent oversell
Product catalog            3   1   2    3    Eventual ok, read-heavy
Search index (ES)          3   1   1    2    Eventual — search can lag
Session/cache (Redis)      1   1   1    2    Eventual — cache is throwaway
Notification status        3   1   1    2    Eventual — email delay ok
```

---

## The Latency vs Consistency Graph

```
Latency
  ▲
  │  ALL ──────────────────────────────────● slowest, strongest
  │
  │  QUORUM ─────────────────────●           sweet spot
  │
  │  LOCAL_QUORUM ────────●                  good for multi-DC
  │
  │  ONE ────●                               fastest, weakest
  │
  └──────────────────────────────────────────────────▶ Availability
             low                                  high
```

---

## Interview Angles 🎯

**Q: What does R + W > N guarantee?**
> It guarantees that the set of nodes written to and the set of nodes read from must overlap by at least one node. That overlapping node always has the latest write, so every read is guaranteed to see the most recent write — strong consistency.

**Q: Cassandra supports tunable consistency — what does that mean practically?**
> Every read and write in Cassandra can specify its own consistency level — ONE, QUORUM, ALL, LOCAL_QUORUM etc. This lets you use strong consistency for critical operations (inventory deduction) and eventual consistency for others (product catalog reads) within the same cluster, without running separate databases.

**Q: If W=2 succeeds but Replica3 missed the write, what happens next time someone reads from Replica3?**
> Two mechanisms fix this. Read Repair: if a quorum read detects a mismatch (Replica3 has old value), it asynchronously writes the correct value back to Replica3 after returning the fresh value to the client. Hinted Handoff: if Replica3 was down during the write, the coordinator stores a hint and delivers it when Replica3 recovers.

**Q: Why is LOCAL_QUORUM preferred over QUORUM in multi-DC deployments?**
> Because global QUORUM requires responses from replicas across all DCs. A cross-DC network hiccup or full DC failure makes the entire cluster unavailable. LOCAL_QUORUM requires a majority only within the local DC, so each DC operates independently. You get consistency within a region without being vulnerable to cross-region latency or failures.

**Q: How does Kafka's `acks=all` + `min.insync.replicas=2` relate to quorum?**
> It's a quorum write on the Kafka log. `acks=all` means the leader waits for all in-sync replicas (ISR) to acknowledge. `min.insync.replicas=2` means at least 2 brokers must be in-sync for the write to succeed. Together they ensure the write is on at least 2 nodes before the producer gets confirmation — a majority write for N=3. If only 1 broker is available, the write fails rather than risk data loss.

---

Say **next** for **Topic 8: Two-Phase Commit (2PC)** — distributed transactions, why they block, and why most modern systems avoid them 🚀
