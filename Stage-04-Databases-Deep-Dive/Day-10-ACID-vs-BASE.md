# Stage 4 — Databases Deep Dive
## Topic 10: ACID vs BASE — Strong vs Eventual Consistency, The Real Trade-off

---

## 1. The Problem That Spawned Both Models

Consistency in distributed systems is fundamentally about one question:

**After a write succeeds, when is it visible to all readers?**

- **Immediately, always:** Strong consistency. Simple to reason about. Expensive to implement across machines.
- **Eventually, given no new writes:** Eventual consistency. Complex to reason about. Scales horizontally with ease.

ACID and BASE are two philosophies built around opposite answers to this question.

---

## 2. ACID — Deep Revisit With Fresh Eyes

We covered ACID mechanics in Topic 1. Now look at the **guarantees from a systems perspective.**

### Atomicity — The All-or-Nothing Contract

```
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 500 WHERE id = 'B';
COMMIT;
```

Either both updates happen or neither does. The database guarantees this via WAL — if the system crashes mid-transaction, the WAL replay undoes partial changes.

**What this means for application code:** You don't need to handle partial failure. No "did the debit succeed but credit fail?" logic. The DB is your safety net.

### Consistency — Invariants Always Hold

Every transaction takes the database from one valid state to another. Constraints (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`) are enforced at commit time.

```sql
-- This will fail atomically — no partial insert
INSERT INTO order_items (order_id, product_id, qty)
VALUES (999, 1, 1);  -- order_id=999 doesn't exist → FK violation → entire transaction rolls back
```

**Important nuance:** The "C" in ACID is partly the DB's job (constraints) and partly the application's job (business logic). The DB enforces structural consistency; you enforce domain consistency.

### Isolation — The Illusion of Serialisation

Concurrent transactions appear to execute sequentially. MVCC makes this practical — readers and writers don't block each other because each transaction sees a consistent snapshot.

**The isolation spectrum — what can go wrong at each level:**

```
Dirty Read:
  T1 writes row (not committed)
  T2 reads T1's uncommitted row  ← reads dirty data
  T1 rolls back
  T2 acted on data that never existed

Non-Repeatable Read:
  T1 reads row: balance = 500
  T2 updates row: balance = 300, commits
  T1 reads same row again: balance = 300  ← different result in same transaction

Phantom Read:
  T1 queries: SELECT * WHERE amount > 100  → 5 rows
  T2 inserts new row with amount = 200, commits
  T1 runs same query again → 6 rows  ← a "phantom" appeared
```

PostgreSQL's default `READ COMMITTED` prevents dirty reads but allows non-repeatable reads. For checkout flows, use `REPEATABLE READ` — price and inventory can't change under you mid-transaction.

### Durability — Committed = Permanent

Once `COMMIT` returns, the data is on durable storage. The WAL is fsynced before COMMIT returns. Even if the OS crashes the next millisecond, the transaction survives.

**The cost:** Every COMMIT triggers an fsync — a flush of the OS buffer cache to physical storage. NVMe SSDs make this fast (~200 microseconds). Spinning disks make this painful (~10ms). This is why storage choice directly impacts PostgreSQL write throughput.

---

## 3. BASE — What It Actually Means

BASE was coined by Eric Brewer (of CAP theorem fame) as the NoSQL counterpart to ACID.

### **B**asically **A**vailable

The system guarantees availability — responses are always returned, even during partial failures. The response might not reflect the most recent write, but the system doesn't return an error or go unavailable.

```
Cassandra node goes down:
  ACID system (requires quorum): might refuse requests until quorum restored
  BASE system: continues serving requests from remaining nodes
               some reads might be stale — that's acceptable
```

### **S**oft State

The state of the system may change over time **even without new input**, as the system converges toward consistency. Old writes propagate to replicas asynchronously. A read at time T might differ from a read at time T+1 for the same key — not because of new writes, but because the replication caught up.

```
Write "price=999" to Cassandra node A at T=0
  T=0ms: Node A: 999, Node B: 750 (old), Node C: 750 (old)
  T=50ms: Node A: 999, Node B: 999 (synced), Node C: 750 (not yet)
  T=100ms: Node A: 999, Node B: 999, Node C: 999 ← converged
```

### **E**ventually Consistent

Given no new writes, all replicas will eventually converge to the same value. The system doesn't guarantee when — but it will happen.

This is not a weak guarantee — for most read scenarios in most applications, "within 100ms" eventual consistency is indistinguishable from strong consistency. The dangerous cases are explicit: reading immediately after writing, or making decisions based on stale reads.

---

## 4. The CAP Theorem — The Forcing Function

Brewer's CAP theorem states that in a distributed system, you can only guarantee **two of three**:

- **C**onsistency: Every read returns the most recent write (or an error)
- **A**vailability: Every request receives a response (not an error)
- **P**artition tolerance: System continues operating despite network partitions

**The catch:** In a real distributed system, **network partitions are inevitable** — a switch fails, a cable gets cut, a cloud AZ goes down. You cannot sacrifice P. So the real choice is **C vs A** during a partition.

```
During a network partition:

CP (choose Consistency):
  Some nodes can't reach others → refuse to serve requests
  → Users get errors, but data is never stale
  → PostgreSQL with synchronous replication, Zookeeper, etcd

AP (choose Availability):
  Continue serving requests even from partitioned nodes
  → Users get responses, but data might be stale
  → Cassandra, DynamoDB, CouchDB
```

**PostgreSQL is CP:** If the primary can't confirm a synchronous replica received the WAL, it can stall. It chooses correctness over availability.

**Cassandra is AP:** With `ONE` consistency, it serves reads from any node regardless of whether other nodes have the latest data. It stays available but might return stale values.

**Important nuance:** CAP is about behavior *during a partition*. During normal operation, both CP and AP systems can be fast and consistent. The difference only manifests under failure.

---

## 5. ACID vs BASE — The Concrete Trade-offs

| Dimension | ACID | BASE |
|---|---|---|
| Consistency | Strong — read always sees latest committed write | Eventual — read may see stale data |
| Availability | Lower — waits for quorum/fsync | Higher — serves from any available node |
| Write latency | Higher — fsync, lock acquisition, 2PC if distributed | Lower — write to one node, replicate async |
| Write throughput | Bounded by single primary | Near-linear horizontal scale |
| Transaction support | Full multi-row, multi-table | Usually single-row or single-partition |
| Failure behaviour | May reject requests to stay correct | Serves stale data rather than erroring |
| Application complexity | Lower — DB handles consistency | Higher — app must handle stale reads, conflicts |
| Schema | Rigid (migration required) | Flexible (schema-on-read) |

---

## 6. Eventual Consistency Patterns — How to Live With It

Choosing BASE doesn't mean giving up on correctness — it means **moving the consistency logic into the application layer.**

### Read-Your-Writes Consistency

After a user writes, guarantee they see their own write — even if other users might see stale data.

```
Write goes to Cassandra node A
User's next read → force to node A (sticky routing)
OR
Write returns a version token → read only from nodes that have this version
```

### Monotonic Reads

A user never sees a value go backward. If they saw version 5, they never see version 3.

```
Implementation: track the highest version seen per user session.
Read from nodes that have at least that version.
```

### Last-Write-Wins (LWW)

When two nodes have conflicting values, the one with the later timestamp wins.

```
Node A: price=999, timestamp=T1
Node B: price=750, timestamp=T2 (T2 > T1)
→ Cassandra resolves conflict: price=750 wins
```

**Risk:** If clocks are skewed between nodes (NTP drift), the "later" timestamp might not actually be the later write. Cassandra uses hybrid logical clocks in newer versions to mitigate this.

### Vector Clocks / CRDTs

Instead of LWW (which discards data), use **CRDTs (Conflict-free Replicated Data Types)** — data structures designed so concurrent updates can always be merged without conflict.

```
Example: Shopping cart as a CRDT (grow-only set)
  Node A: {item1, item2}
  Node B: {item1, item3}  (added item3 concurrently)
  Merged: {item1, item2, item3}  ← union, no conflict

Removes need a tombstone:
  Remove item1 → {item1: deleted, item2, item3}
  Merge: tombstone wins
```

DynamoDB uses vector clocks. Riak built its entire model on CRDTs.

---

## 7. The Real Choice — Not SQL vs NoSQL

The ACID vs BASE trade-off is **independent of SQL vs NoSQL syntax.** The actual dimensions:

```
What you're really choosing:

Strong Consistency ←──────────────────────→ Eventual Consistency
      │                                              │
PostgreSQL                                     Cassandra
MySQL InnoDB                                   DynamoDB
Oracle                                         CouchDB
      │                                              │
MongoDB (readConcern: majority)            MongoDB (readConcern: local)
Cosmos DB (Strong)                         Cosmos DB (Eventual)
```

Some systems are tunable — Cassandra lets you choose per-query consistency. MongoDB lets you choose read/write concern. The database is a tool; you choose the consistency level appropriate to each operation.

---

## 8. Which Operations Need ACID, Which Can Use BASE

This is the practical skill — applying the right model to the right operation.

| Operation | Model | Reason |
|---|---|---|
| Payment debit/credit | ACID | Money cannot be partially transferred |
| Inventory decrement during checkout | ACID | Overselling is a real loss |
| User password change | ACID | Auth state must be consistent immediately |
| Product catalog browse | BASE | Stale product description by 100ms is fine |
| Cart update | BASE | Cart is ephemeral; eventual sync is fine |
| Social media like count | BASE | "1,284 likes" vs "1,285 likes" — irrelevant |
| Order history display | BASE | Seeing an order from 1 second ago is acceptable |
| Fraud detection rule update | ACID | All nodes must use the same rules simultaneously |
| Analytics event ingestion | BASE | Eventual aggregation is fine for reports |
| Flash sale inventory | ACID | Must prevent overselling under high concurrency |

---

## 9. ShopSphere Mapping

```
Service          Operation                    Model     Why
─────────────────────────────────────────────────────────────────────
Payment Svc      Debit + Credit               ACID      Cannot lose money
Order Svc        Place order (inventory--)    ACID      Oversell prevention
User Svc         Registration, login          ACID      Auth correctness
Inventory Svc    Reserve stock                ACID      Prevent double booking

Product Svc      Browse catalog               BASE      Stale by seconds: fine
Search Svc       Search results               BASE      Derived from Kafka; eventual
Cart Svc         Add/remove item              BASE      Redis; eventual persistence
Analytics Svc    Event ingestion              BASE      Cassandra; eventual aggregate
Notification Svc Send email/push              BASE      At-least-once, idempotent
Review Svc       Display ratings              BASE      "4.2★" vs "4.3★": irrelevant
```

**The checkout flow — mixing both:**

```java
@Transactional  // ACID boundary
public Order placeOrder(UUID userId, CartRequest cart) {
    // 1. Read current inventory — ACID (from primary, current snapshot)
    Inventory inv = inventoryRepo.findByProductIdWithLock(cart.getProductId());

    // 2. Decrement inventory — ACID
    if (inv.getQty() < cart.getQty()) throw new InsufficientStockException();
    inv.setQty(inv.getQty() - cart.getQty());
    inventoryRepo.save(inv);

    // 3. Create order — ACID
    Order order = orderRepo.save(new Order(userId, cart));

    // 4. Publish event to Kafka — BASE (async, outside transaction)
    // Done via Outbox pattern — event written to outbox table inside same ACID transaction
    outboxRepo.save(new OutboxEvent("ORDER_PLACED", order));

    return order;  // COMMIT here — ACID guarantees both inventory and order persisted
}
// After commit: Outbox relay picks up event → Kafka → Notification, Analytics (BASE)
```

The ACID part: inventory decrement + order creation — atomic, consistent, isolated, durable.
The BASE part: notification, analytics, search index update — eventual, via Kafka.

---

## 10. Interview Q&A

**Q: What does "eventually consistent" mean in practice? How eventual is eventual?**
A: It means given no new writes, all replicas will converge to the same value — but without a time bound guarantee. In practice for most systems: milliseconds to seconds for in-datacenter replication, seconds to minutes across regions. For most product features (catalog, reviews, feeds), this is indistinguishable from strong consistency in user experience. The dangerous cases are explicit: reading immediately after writing your own data (read-your-writes), and making irreversible decisions (payments) on potentially stale data.

**Q: Why can't you just use ACID everywhere and avoid dealing with eventual consistency?**
A: ACID requires coordination — fsync for durability, locks for isolation, quorum for distributed consistency. Each of these adds latency and caps write throughput to what one machine (or tightly coordinated cluster) can handle. For globally distributed systems serving millions of writes/second, strong consistency becomes a bottleneck. Cassandra at Uber processes millions of trips per second — ACID would require a single master handling all writes, which physically cannot sustain that throughput. BASE is not a compromise but a deliberate choice for specific access patterns.

**Q: What is the CAP theorem and what does it mean for your system design?**
A: CAP states a distributed system can guarantee at most two of: Consistency (reads always return latest write), Availability (requests always get responses), Partition Tolerance (system works despite network partitions). Since network partitions are inevitable in production, the real choice is C vs A during a partition. CP systems (PostgreSQL, ZooKeeper) refuse requests rather than serve stale data — correct but potentially unavailable. AP systems (Cassandra, DynamoDB) serve from any available node — always available but potentially stale. The choice depends on whether an error or stale data is more harmful for your use case.

**Q: How does Cassandra handle conflicting writes to the same row from two different nodes?**
A: By default, Cassandra uses Last-Write-Wins (LWW) based on the write timestamp. The write with the higher timestamp overwrites the other. Risk: if node clocks are skewed (NTP drift), the "later" timestamp might not be the actually later write — data can be silently overwritten. For scenarios where concurrent writes must all be preserved, use CRDTs (Counters in Cassandra are a built-in CRDT) or application-level conflict resolution with vector clocks.

**Q: You're designing a flash sale for ShopSphere — 10,000 users competing for 100 units simultaneously. ACID or BASE?**
A: ACID, specifically with pessimistic locking or optimistic concurrency control. Flash sale inventory is the classic oversell problem — if two transactions read qty=1 concurrently and both decrement, you get qty=-1 (sold the same unit twice). Solution: `SELECT ... FOR UPDATE` (pessimistic lock) to serialize access, or optimistic locking with a version column (`UPDATE inventory SET qty = qty - 1, version = version + 1 WHERE id = ? AND version = ? AND qty >= 1` — fails if someone else changed it first). BASE/eventual consistency would allow both transactions to see qty=1 and both succeed — unacceptable for inventory.

---

Ready for **Topic 11: Schema Evolution — adding/removing columns safely, backward compatibility, rolling migrations**?
