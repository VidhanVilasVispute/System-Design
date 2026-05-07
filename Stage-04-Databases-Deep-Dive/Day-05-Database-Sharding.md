# Stage 4 — Databases Deep Dive
## Topic 5: Database Sharding — Horizontal Partitioning, Shard Keys, Hot Shards, Resharding

---

## 1. What Sharding Actually Is

Sharding is **horizontal partitioning across multiple independent database machines**. Each machine (shard) owns a subset of the data and is a fully independent PostgreSQL instance — its own CPU, memory, disk, connections.

Contrast with what we've covered:

| Technique | What it does | Still one machine? |
|---|---|---|
| Indexing | Faster reads on same data | ✅ Yes |
| Partitioning | Splits table into physical chunks | ✅ Yes |
| Read Replicas | Copies data to other machines for reads | ✅ Writes still one machine |
| **Sharding** | Splits data across machines for reads AND writes | ❌ No |

Sharding is the last resort — you reach for it when:
- A single primary can no longer handle write throughput
- Dataset size exceeds what one machine can store
- Read replicas are saturated and adding more isn't enough

---

## 2. How Sharding Works

You pick a **shard key** — a column whose value determines which shard a row lives on. Every read and write must include the shard key so the application (or middleware) knows which machine to talk to.

```
ShopSphere Orders:

Shard 0 (DB machine 0): user_ids 0–24M
Shard 1 (DB machine 1): user_ids 25M–49M
Shard 2 (DB machine 2): user_ids 50M–74M
Shard 3 (DB machine 3): user_ids 75M–100M
```

Application layer:

```java
public DataSource getShardForUser(UUID userId) {
    int shardIndex = Math.abs(userId.hashCode()) % TOTAL_SHARDS;
    return shardDataSources[shardIndex];
}
```

Every query for `user_id = X` goes to exactly one shard. No coordination needed.

---

## 3. Shard Key Selection — The Most Critical Decision

The shard key determines everything: data distribution, query patterns, hotspots, and how painful resharding will be. You cannot change it without rewriting the entire system.

### Properties of a Good Shard Key

**High cardinality:** Enough distinct values to spread across shards evenly. `user_id` (UUID) — excellent. `country_code` — terrible (only ~200 values, most traffic in US/IN).

**Even distribution:** Values should hash evenly across shards. Sequential IDs with range-based sharding cause hotspots (all new writes go to the last shard).

**Query locality:** The key you shard on should be the key you filter on most. If you shard by `user_id` but frequently query `WHERE product_id = X`, you hit every shard for that query (scatter-gather).

**Immutable:** Once assigned, the shard key for a row must never change. If a user's `user_id` changes (it shouldn't, but hypothetically), you'd need to move the row to a different shard — expensive.

### Common Shard Key Choices

| Entity | Shard Key | Why |
|---|---|---|
| Orders | `user_id` | Orders are almost always queried per user |
| Messages | `conversation_id` | All messages in a thread stay on one shard |
| Products | `seller_id` | Seller's catalog queries stay local |
| Events/Logs | `tenant_id` (B2B) | All tenant data on one shard |

### Bad Shard Key Examples

**Timestamp:** All current writes go to the shard holding "today" — severe write hotspot.

**Status:** Low cardinality, uneven distribution (most orders are DELIVERED, few are PENDING).

**Product category:** Too few categories, uneven traffic (electronics >> stationery).

---

## 4. Sharding Strategies

### Range-Based Sharding

Assign ranges of shard key values to each shard.

```
Shard 0: user_id 1       → 25,000,000
Shard 1: user_id 25M+1   → 50,000,000
Shard 2: user_id 50M+1   → 75,000,000
Shard 3: user_id 75M+1   → 100,000,000
```

**Pro:** Range queries within a shard key are efficient (all orders for users 1–25M are on Shard 0).

**Con:** New users are always sequential → new writes hammer the last shard. Classic write hotspot.

### Hash-Based Sharding

Apply a hash function to the shard key, mod by shard count.

```
shard_index = hash(user_id) % num_shards
```

**Pro:** Uniform distribution — hash functions spread values evenly.

**Con:** Range queries are destroyed. `WHERE user_id BETWEEN 1000 AND 2000` hits all shards (scatter-gather). Also, adding a shard changes `% num_shards` — everything remaps.

### Directory-Based Sharding

A lookup table maps shard key → shard.

```
Lookup Service:
user_id 1       → Shard 2
user_id 2       → Shard 0
user_id 3       → Shard 1
...
```

**Pro:** Full flexibility — move specific users between shards as needed. Can implement custom placement logic.

**Con:** Lookup service is a single point of failure and a performance bottleneck. Must be cached aggressively.

---

## 5. The Hot Shard Problem

A hot shard is one that receives **disproportionately more traffic** than others, becoming a bottleneck while other shards sit idle.

### How It Happens

**Celebrity problem:** Shard by `user_id`. A seller with 10M followers places a flash sale. All product queries for that seller's items hit the same shard. Other shards are at 10% CPU. That shard is at 100%.

**Range-based temporal hotspot:** Shard orders by date range. All new orders go to today's shard. The "current" shard is always hot; older shards are cold.

**Skewed data distribution:** Shard by `country_code`. India and US have 80% of users. Those two shards are overloaded; others are nearly empty.

### Mitigations

**Shard splitting:** When a shard gets too hot, split it into two shards. Half the data moves to a new machine. Requires a migration window and careful routing update.

**Adding a compound shard key:** Instead of `user_id` alone, use `(user_id, created_at_bucket)`. Spreads a single user's data across time-bucketed shards.

**Application-level caching:** For celebrity data, absorb reads in Redis before they hit the DB shard. Hot shard problem is often a read problem — cache the hot data.

**Consistent hashing:** Reduces hot shard risk during topology changes (covered in the next topic).

---

## 6. Cross-Shard Operations — The Pain

Sharding breaks all the things RDBMS gives you for free.

### Cross-Shard Joins

```sql
-- This works on a single DB:
SELECT o.order_id, u.email
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2024-01-01';
```

If `orders` is sharded by `user_id` and `users` is on a separate (unsharded) DB or different shard — this join is impossible at the DB level. You must:

1. Query orders from multiple shards in parallel
2. Collect all `user_id`s
3. Query users by those IDs
4. Join in application memory

This is called **scatter-gather + application-side join**. Expensive. Slow. But sometimes unavoidable.

### Cross-Shard Transactions

A transaction spanning two shards requires distributed 2PC (Two-Phase Commit). Most teams avoid this entirely — instead they:
- Design data models so transactions stay within one shard
- Use the Saga pattern (compensating transactions) for cross-shard workflows
- Accept eventual consistency for cross-shard operations

### Global Queries

`SELECT COUNT(*) FROM orders` → must query all shards, sum results. Every aggregation, every report, every admin dashboard query becomes a scatter-gather operation. This is why sharded systems typically maintain a **separate analytics DB** (data warehouse) that aggregates from all shards.

---

## 7. Resharding — The Expensive Nightmare

You start with 4 shards. Two years later, 4 isn't enough. You need 8.

With hash-based sharding (`hash(user_id) % 4` → `hash(user_id) % 8`), **every single row remaps**. A user that was on Shard 2 might now belong on Shard 5. You must move approximately **half the data** across the network.

### The Resharding Process

1. **Provision new shards** — spin up new DB machines
2. **Dual-write period** — write to both old and new shard topology simultaneously
3. **Backfill** — copy existing data to new shards in the background (slow, weeks potentially)
4. **Verify** — checksums confirm data integrity
5. **Cutover** — route reads to new topology
6. **Cleanup** — delete data from old shards

**Risk during resharding:**
- Data inconsistency during backfill window
- Increased write latency (dual-write overhead)
- Potential for missed writes if dual-write logic has bugs
- The backfill job competes for I/O with live traffic

This is why the initial shard key choice is so critical — resharding is a multi-week engineering project that keeps the team up at night.

**Consistent hashing** (next topic) was specifically invented to minimize resharding cost.

---

## 8. When NOT to Shard

Sharding adds enormous operational complexity. Before sharding, exhaust:

1. **Vertical scaling** — bigger machine (more RAM, faster NVMe). PostgreSQL on a 64-core, 512GB RAM machine with NVMe handles enormous workloads.
2. **Read replicas** — most web apps are 80%+ reads. Scale reads first.
3. **Caching** — Redis in front of your DB absorbs the majority of read load.
4. **Partitioning** — reduces index size, enables pruning, enables fast drops of old data.
5. **Query optimization** — often a missing index or bad query plan is the real bottleneck.

Shard when you've genuinely exhausted all of the above and write throughput or data volume is the bottleneck.

---

## 9. ShopSphere Mapping

ShopSphere at current scale doesn't need sharding — but here's the design for when it does:

**Orders sharded by `user_id`:**
- All of a user's orders on one shard → `getOrderHistory(userId)` hits one shard
- Order placement writes to one shard → no cross-shard transaction needed
- Cross-shard: admin report "all orders today" → scatter-gather across all shards → aggregate in app

**Products NOT sharded — kept on a replicated primary:**
- Product catalog is read-heavy, write-light
- Read replicas handle the load
- If sharded, product lookup during order placement would require cross-shard join with orders

**Separate analytics pipeline:**
- Kafka streams all order events → Flink/Spark aggregates → writes to data warehouse
- Admin reports query the warehouse, not the shards
- Eliminates scatter-gather from production shards

```
[Order Service]
      │ write (user_id determines shard)
      ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │
│ users   │  │ users   │  │ users   │  │ users   │
│ 0–25M   │  │ 25–50M  │  │ 50–75M  │  │ 75–100M │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
      │            │            │            │
      └────────────┴────────────┴────────────┘
                          │
                     Kafka topic
                          │
                   Analytics Warehouse
```

---

## 10. Interview Q&A

**Q: How do you choose a shard key?**
A: Three criteria: high cardinality (enough distinct values to distribute evenly), query locality (the key you shard on should be the key you filter on most), and immutability (can't change after assignment). Typically the entity's primary identifier — `user_id` for user-centric data, `tenant_id` for B2B systems, `conversation_id` for messaging.

**Q: What is a hot shard and how do you fix it?**
A: A hot shard receives disproportionate traffic — often from a celebrity user, a skewed key distribution, or temporal patterns. Fix: for read hotspots, add a caching layer (Redis) in front of the shard. For write hotspots, split the shard (one shard becomes two) or introduce a compound shard key to spread writes further. For temporal hotspots, avoid time-based range sharding — use hash-based sharding instead.

**Q: How do you handle a transaction that spans two shards?**
A: Ideally, redesign the data model so the transaction stays within one shard — put all entities that transact together under the same shard key. If unavoidable, use the Saga pattern: break the transaction into local steps with compensating transactions for rollback. Avoid distributed 2PC — it's slow, has failure modes, and kills throughput.

**Q: What's the difference between partitioning and sharding?**
A: Partitioning is a single-machine optimization — one logical table split into physical segments on the same server. The application sees one table; the DB engine handles routing to partitions internally. Sharding is multi-machine — each shard is an independent DB server. The application must know which shard to target. Sharding scales writes and storage across machines; partitioning doesn't.

**Q: A company shards their user table by `user_id` but now needs to query "all users in a given city." How?**
A: This is a scatter-gather query — city doesn't map to a shard, so you must query all shards in parallel and aggregate results in the application layer. For frequent such queries, maintain a secondary index in a separate system: either a dedicated lookup DB (maps city → user_ids) or index into Elasticsearch which queries all shards asynchronously and returns results. Alternatively, use directory-based sharding with a lookup service that can answer city-based queries.

---

Ready for **Topic 6: Consistent Hashing — how it assigns data to nodes and why reshuffling is minimal when nodes join or leave**?
