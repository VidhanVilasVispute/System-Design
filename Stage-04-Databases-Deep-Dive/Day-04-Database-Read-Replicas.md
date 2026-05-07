# Stage 4 — Databases Deep Dive
## Topic 4: Database Read Replicas — Internals, Consistency Trade-offs, Failure Modes

---

## 1. Why Read Replicas Exist

A write-heavy production DB eventually hits a ceiling — not on writes, but on **reads competing with writes** for I/O, CPU, and locks.

In most web applications, the read/write ratio is heavily skewed:

```
ShopSphere traffic pattern:
- Product browse:     ~70% of all queries  → reads
- Search:             ~15%                 → reads
- Order placement:     ~5%                 → writes
- Admin/analytics:    ~10%                 → heavy reads
```

Running analytics on your primary while users are checking out is dangerous — a full table scan holds I/O bandwidth, evicts hot pages from `shared_buffers`, and can cause query plan contention. Read replicas isolate this.

---

## 2. How Replication Works Internally — WAL Shipping

PostgreSQL replication is built entirely on the **Write-Ahead Log (WAL)**.

### Physical Replication (Streaming Replication — Default)

Every change in PostgreSQL is first written as a WAL record. WAL records are **physical** — they describe exact byte-level changes to data pages.

```
Primary:
  User commits transaction
       ↓
  WAL record written: "Page 1847, offset 32–64, new bytes: [...]"
       ↓
  WAL sender process streams record to replica
       ↓
Replica:
  WAL receiver process receives record
       ↓
  WAL record applied: overwrite page 1847, offset 32–64
       ↓
  Replica's data files now match primary
```

The replica is a **byte-for-byte copy** of the primary. It can't accept writes (read-only). It can't even run arbitrary SQL during recovery — it's replaying the WAL.

### Replication Slots

Without replication slots, if a replica falls behind and the primary recycles (deletes) old WAL segments, the replica can never catch up — it's permanently broken.

**Replication slots** make the primary keep WAL segments until every slot has consumed them:

```sql
-- On primary:
SELECT * FROM pg_replication_slots;
-- name            | active | restart_lsn
-- replica1_slot   | t      | 0/3A21F8
```

**Risk:** If a replica dies and you forget to drop its slot, the primary keeps accumulating WAL forever → disk fills → primary crashes. Always monitor slot lag.

---

## 3. Synchronous vs Asynchronous Replication

This is the **core trade-off** for read replicas.

### Asynchronous (Default)

```
Client → COMMIT → Primary writes WAL → returns "OK" to client
                       ↓ (background)
                  streams WAL to replica
```

Primary doesn't wait for replica acknowledgment. COMMIT returns in microseconds. Replica applies WAL independently — typically milliseconds behind, but can be seconds under load.

**Risk:** If primary crashes after COMMIT but before replica receives WAL → **data loss**. That transaction is gone.

### Synchronous

```sql
-- On primary:
ALTER SYSTEM SET synchronous_standby_names = 'replica1';
```

```
Client → COMMIT → Primary writes WAL
                       ↓
               waits for replica1 to confirm WAL receipt
                       ↓ (replica acknowledges)
               returns "OK" to client
```

Primary blocks COMMIT until at least one replica confirms it has the WAL. **Zero data loss** (RPO = 0). Cost: every write now has a network round trip in its critical path — adds 1–5ms latency to every transaction.

### `synchronous_commit` Settings (PostgreSQL)

| Setting | Behaviour | RPO | Latency |
|---|---|---|---|
| `off` | Don't even wait for local WAL fsync | Data loss possible | Fastest |
| `local` | Wait for local WAL fsync only | Local durable, replica may lose | Fast |
| `remote_write` | Replica received + written to OS buffer | Near-zero loss | Medium |
| `remote_apply` | Replica applied the change | Zero loss | Slowest |
| `on` (default) | Same as `remote_write` if sync standby set | — | — |

ShopSphere strategy:
- Payment DB → `synchronous_commit = remote_apply` (no data loss ever)
- Product/Order read replicas → `asynchronous` (few ms lag acceptable)

---

## 4. Replication Lag — The Real Consistency Problem

Lag = how far behind the replica is from the primary, measured in bytes of WAL not yet applied or in time.

```sql
-- On primary, check replica lag:
SELECT
  client_addr,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  (sent_lsn - replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### What Causes Lag

**Write storms on primary:** A bulk `UPDATE` on 5M rows generates enormous WAL. Replica processes WAL single-threaded (prior to PG 16) — can't keep up with a flood.

**Long-running queries on replica:** If a replica is executing a slow analytics query, WAL application is blocked by lock contention on that table. Lag accumulates.

**Network bandwidth saturation:** WAL is streamed over the network. Heavy replication on a congested network → lag spikes.

**Replica I/O bottleneck:** Replica disk can't write WAL as fast as primary generates it.

### Handling Lag in Application Code

**Pattern 1: Primary read for critical paths**

```java
// Checkout — must read current inventory from primary
@Transactional  // non-readOnly → routes to primary
public CheckoutResult checkout(UUID userId, CartRequest cart) {
    Inventory inv = inventoryRepo.findById(cart.getProductId()); // reads primary
    // ...
}
```

**Pattern 2: Read-your-writes via LSN tracking**

After a write, capture the WAL LSN (Log Sequence Number). Pass it to the client as a token. On subsequent reads, only serve from a replica that has replayed up to that LSN.

```sql
-- After commit, get current LSN:
SELECT pg_current_wal_lsn();  -- e.g., 0/3A21F8

-- On replica, check if it's caught up:
SELECT pg_last_wal_replay_lsn() >= '0/3A21F8';
```

If replica hasn't caught up → fall back to primary for that read.

**Pattern 3: Tolerate lag explicitly**

Product catalog, search results, review counts — none of these need real-time consistency. Design these features to explicitly tolerate stale reads. Document the SLO: "Product availability shown is accurate within 5 seconds."

---

## 5. Failure Modes and Failover

### Primary Failure — Promotion

When the primary dies:

1. Monitoring detects primary is unreachable (health check timeout)
2. **Patroni / repmgr / AWS RDS** elects a replica as new primary
3. Elected replica is **promoted** — it stops receiving WAL and starts accepting writes
4. Other replicas reconfigure to follow the new primary
5. Application's connection string (DNS or load balancer) updates to point to new primary

**Promotion lag:** The promoted replica may not have the latest WAL from the old primary. Any transactions committed on the old primary after the last WAL received by this replica are **lost**. This is the RPO (Recovery Point Objective) — minimized by synchronous replication.

### Split-Brain

If network partitions the primary from replicas but primary is still alive — both primary and promoted replica start accepting writes. Data diverges.

**Prevention:** Use a **fencing mechanism** — when a replica is promoted, send a STONITH (Shoot The Other Node In The Head) signal to the old primary via a separate network path, or revoke its storage access.

### Replica Falling Too Far Behind

If a replica lags so much that the primary has already recycled the WAL segments it needs (and no replication slot is protecting them), the replica can't continue streaming. It must be rebuilt from a base backup.

---

## 6. Read Replica Topologies

### Single Replica

```
Primary → Replica
```
Simple. Doubles read capacity. Single point of replica failure.

### Multiple Replicas Behind Load Balancer

```
Primary → Replica 1 ─┐
         → Replica 2 ─┤→ Read Load Balancer → App reads
         → Replica 3 ─┘
```

PgBouncer or HAProxy in front of replicas. Round-robin read distribution. If one replica lags, route away from it based on lag monitoring.

### Cascading Replication

```
Primary → Replica 1 (sync, same DC, hot standby)
                ↓
         Replica 2 (async, different region, analytics)
```

Replica 1 is the synchronous hot standby for failover. Replica 2 is a downstream replica — it replicates from Replica 1, not the primary. This offloads WAL streaming cost from the primary.

### Dedicated Analytics Replica

Heavy analytical queries (aggregations, reports) run separate from user-facing read replicas. This replica might have different indexes optimised for analytical patterns, and can lag minutes rather than milliseconds — that's acceptable for dashboards.

---

## 7. ShopSphere Mapping

```
                        ┌─────────────────────────────────────┐
                        │         PostgreSQL Primary           │
                        │  (writes: orders, payments, users)   │
                        └──────────────┬──────────────────────┘
                                       │ WAL streaming
                    ┌──────────────────┼──────────────────────┐
                    ▼                  ▼                       ▼
             Replica 1           Replica 2              Replica 3
         (sync hot standby)  (async, user reads)    (async, analytics)
         same AZ, failover   order history, catalog  admin dashboard
         target              product browse          reports
```

**Service routing:**

| Service | Operation | Target |
|---|---|---|
| Order Service | `placeOrder()` | Primary |
| Order Service | `getOrderHistory()` | Replica 2 |
| Product Service | `getProductDetails()` | Replica 2 |
| Inventory Service | `checkStock()` during checkout | Primary |
| Admin Service | sales reports | Replica 3 |
| Payment Service | all operations | Primary (sync replica as standby) |

**Spring Boot implementation:**

```java
@Configuration
public class DataSourceConfig {

    @Bean @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() { return DataSourceBuilder.create().build(); }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() { return DataSourceBuilder.create().build(); }

    @Bean
    public DataSource routingDataSource() {
        Map<Object, Object> targets = new HashMap<>();
        targets.put("primary", primaryDataSource());
        targets.put("replica", replicaDataSource());

        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                    ? "replica" : "primary";
            }
        };
        routing.setTargetDataSources(targets);
        routing.setDefaultTargetDataSource(primaryDataSource());
        return routing;
    }
}
```

---

## 8. Interview Q&A

**Q: What is replication lag and what are its consequences?**
A: Replication lag is the delay between a write being committed on the primary and being visible on a replica, measured in bytes of WAL backlog or wall-clock time. Consequences: stale reads — a user who just placed an order might not see it when routed to a lagging replica. For critical reads (inventory, payments), always read from primary. For non-critical reads, design the feature to explicitly tolerate a defined staleness window.

**Q: How does PostgreSQL streaming replication differ from logical replication?**
A: Streaming (physical) replication ships raw WAL bytes — exact page-level changes. The replica is a binary copy of the primary and is read-only. Logical replication decodes WAL into row-level changes (INSERT/UPDATE/DELETE) and can replicate to a different PG version, a different table structure, or even a non-PG system. Logical replication allows the target to accept writes, enabling selective replication of specific tables. Trade-off: logical replication has more overhead and doesn't replicate DDL automatically.

**Q: A user places an order and immediately opens "My Orders" — it shows nothing. What happened and how do you fix it?**
A: Read-your-writes violation — the order write went to primary, but the subsequent read hit an asynchronous replica that hadn't yet received that WAL. Fix options: (1) Route the user's reads to primary for a short window after a write. (2) Capture the WAL LSN after the write, pass it to the client as a consistency token, and only serve reads from replicas that have replayed up to that LSN. (3) For this specific pattern, always read order history from primary — it's a low-volume, high-importance read.

**Q: What happens if you promote a replica and the old primary comes back online?**
A: Split-brain risk — the old primary doesn't know it was demoted and tries to resume writes. You get two primaries diverging data. Prevention: fencing — before promotion completes, ensure the old primary is fenced off (storage revoked, network isolated, or STONITH signal sent). Tools like Patroni handle this automatically using a distributed lock (etcd or Consul) to ensure only one node believes it's primary.

**Q: When would you choose synchronous over asynchronous replication?**
A: Synchronous when RPO = 0 is a hard requirement — financial transactions, audit records, anything where data loss is unacceptable. Asynchronous when write latency matters more than zero data loss — product catalog, session data, logs. Hybrid: use one synchronous replica for failover safety and additional async replicas for read scaling.

---

Ready for **Topic 5: Database Sharding — horizontal partitioning, shard keys, hot shard problem, resharding cost**?
