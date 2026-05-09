# Stage 4 — Databases Deep Dive
## Topic 12: LSM-Tree vs B-Tree — Storage Engine Internals, Write vs Read Optimisation

---

## 1. The Fundamental Problem Both Solve

Every database must answer: **where on disk does a row live, and how do we find it fast?**

The answer is the storage engine — the lowest layer of the database, responsible for:
- How data is laid out on disk
- How writes are applied
- How reads find data
- How deletes and updates are handled

B-Tree and LSM-Tree are two fundamentally different answers to this problem, with opposite trade-offs.

---

## 2. B-Tree — Read-Optimised Storage

### Structure

A B-Tree index file is divided into fixed-size **pages** (typically 8KB in PostgreSQL, 16KB in MySQL). The tree is stored across these pages on disk.

```
Page Layout:
┌─────────────────────────────────────┐
│ Header (page type, free space ptr)  │
│ Keys: [10 | 25 | 40 | 67 | 89]     │
│ Pointers: [p1|p2|p3|p4|p5|p6]      │
│ Free space                          │
└─────────────────────────────────────┘
```

Each internal node holds keys and child pointers. Leaf nodes hold keys and row pointers (ctid in PostgreSQL, primary key in MySQL secondary indexes).

### How a Write Works in B-Tree

```
INSERT INTO orders (id, user_id, amount) VALUES (501, 42, 999);
```

1. Traverse B-Tree to find the correct leaf page for `id=501`
2. **Read that page from disk into memory** (if not already in buffer pool)
3. Insert the new key into the page
4. **Write the modified page back to disk** (random write — page 1847)
5. If the page is full → **page split**: allocate new page, redistribute keys, update parent → cascade upward
6. WAL record written for crash recovery

**The killer:** Step 4 is a **random write** — you must seek to page 1847 on disk and overwrite it. On spinning disks, a random seek takes 5–10ms. Even on SSDs, random writes cause write amplification and wear.

Page splits make this worse — a single insert can cascade into multiple random writes up the tree.

### How a Read Works in B-Tree

```
SELECT * FROM orders WHERE id = 501;
```

1. Traverse B-Tree from root to leaf — O(log N) page reads
2. Each page read is potentially a random disk access
3. Leaf contains pointer to heap page → one more random read

For a 50M row table with 8KB pages, tree height is ~4. Four sequential page reads, then the heap fetch = 5 total I/Os. **Predictably fast.**

**This is why B-Tree wins for reads** — the structure is permanently sorted. Finding any key is always O(log N) with no indirection layers.

---

## 3. LSM-Tree — Write-Optimised Storage

LSM-Tree (Log-Structured Merge Tree) was designed by Patrick O'Neil et al. in 1996 specifically to eliminate random writes.

**Core insight:** Sequential writes are 10–100x faster than random writes on any storage medium (spinning disk or SSD). If we can make all writes sequential, write throughput explodes.

### The Three-Layer Structure

```
Layer 1: WAL (Write-Ahead Log)         — sequential, on disk
Layer 2: Memtable                      — in memory, sorted
Layer 3: SSTables (Sorted String Tables) — on disk, immutable, sorted
```

### How a Write Works in LSM-Tree

```
INSERT INTO order_events (user_id, event_time, event_type) 
VALUES ('u42', now(), 'PLACED');
```

1. **Write to WAL** — sequential append to commit log on disk. Fast. For crash recovery.
2. **Write to Memtable** — insert into in-memory sorted structure (typically a Red-Black tree or skip list). Sub-microsecond.
3. **Return success to client** ← this is it. Done.

No disk seek. No page lookup. No random write. The write is complete in step 2.

```
Write path:
Client → WAL (sequential append) → Memtable (in-memory) → ACK
```

**This is why Cassandra sustains millions of writes/second** — every write is a memory operation plus a sequential disk append.

### Memtable Flush — Memtable → SSTable

When the Memtable reaches a size threshold (~64MB in Cassandra):

1. Memtable is frozen — no more writes to it
2. A new Memtable takes over for incoming writes
3. Frozen Memtable is flushed to disk as an **SSTable** — an immutable, sorted file

```
Memtable (sorted in memory):
  (u10, T=100, PLACED)
  (u42, T=105, PLACED)
  (u42, T=110, CONFIRMED)
  (u99, T=107, PLACED)
       ↓ flush
SSTable file on disk (sorted, immutable):
  [u10|T=100|PLACED][u42|T=105|PLACED][u42|T=110|CONFIRMED][u99|T=107|PLACED]
```

The flush is a **sequential write** — write the entire sorted Memtable to disk in one pass. Fast, no random seeks.

### How a Read Works in LSM-Tree

```
SELECT * FROM order_events WHERE user_id = 'u42';
```

1. Check Memtable (most recent data, in memory)
2. Check each SSTable on disk, newest first
3. Merge results, handle tombstones (deletes), return

**Problem:** Data for `u42` might be spread across 10 SSTables accumulated over time. Each SSTable requires a disk read. **Read amplification** — one logical read requires multiple physical reads.

**Mitigation 1 — Bloom Filters:**
Each SSTable has a Bloom filter — a probabilistic data structure that answers "is key X in this SSTable?" in O(1) with no disk access. If the Bloom filter says no, skip the SSTable entirely. This eliminates most SSTable reads.

```
Read with Bloom filters:
  u42 in Memtable? No
  u42 in SSTable 10? Bloom: YES → read it
  u42 in SSTable 9?  Bloom: NO  → skip
  u42 in SSTable 8?  Bloom: YES → read it
  u42 in SSTable 7?  Bloom: NO  → skip
  ...
Result: only 2 SSTables actually read instead of 10
```

**Mitigation 2 — Sparse Index per SSTable:**
Each SSTable has a small index (first key of every N-th block). Used to binary-search within the SSTable without reading all of it.

**Mitigation 3 — Compaction:**

---

## 4. Compaction — The Heart of LSM-Tree

Over time, SSTables accumulate. Reads get slower as more SSTables must be checked. Deleted data (tombstones) takes space. Updated data has multiple versions spread across SSTables.

**Compaction** is a background process that merges SSTables:

```
Before compaction:
  SSTable 1: (u42, T=100, PLACED), (u42, T=110, CONFIRMED)
  SSTable 2: (u10, T=120, PLACED), (u42, T=130, SHIPPED)
  SSTable 3: (u42, T=140, DELIVERED), tombstone(u10, T=120)

After compaction (merge + sort + deduplicate):
  SSTable merged: (u42, T=100, PLACED), (u42, T=110, CONFIRMED),
                  (u42, T=130, SHIPPED), (u42, T=140, DELIVERED)
  (u10 row deleted — tombstone applied)
```

Compaction is entirely sequential I/O — read SSTables, merge in memory, write new SSTable. Fast on any storage.

### Compaction Strategies (Cassandra)

**Size-Tiered Compaction (STCS) — default:**
When N SSTables of similar size accumulate, merge them into one larger SSTable. Write-optimised — minimal compaction overhead. Read-amplification grows over time until compaction runs.

**Levelled Compaction (LCS):**
SSTables are organized in levels (L0, L1, L2...). Each level is 10x larger than the previous. L1+ SSTables have no key overlap — a key exists in exactly one SSTable per level. Read performance much better (guaranteed few SSTables to check). Write amplification higher — more compaction work.

**Time-Window Compaction (TWCS):**
Groups SSTables by time window. Designed for time-series data — compact within a window, never across windows. When a window is past retention period, drop entire SSTable file (instead of row-by-row delete). Perfect for Cassandra time-series use cases in ShopSphere.

---

## 5. Deletes — Tombstones

In B-Tree: delete a row → mark the slot free in the page. Simple.

In LSM-Tree: SSTables are **immutable**. You can't delete from them.

**Solution: tombstones.** A delete writes a special marker:

```
DELETE FROM order_events WHERE user_id = 'u10' AND event_time = T120;
→ Writes tombstone: (u10, T120, DELETED) into Memtable
→ Flushed to SSTable as a tombstone record
```

During reads: if you encounter a tombstone, suppress the underlying data. During compaction: tombstone + original record merge → both are discarded.

**Tombstone problem:** If compaction hasn't run recently, reads must process many tombstones before finding actual data. Heavy delete workloads need aggressive compaction tuning.

---

## 6. Side-by-Side Comparison

| Property | B-Tree | LSM-Tree |
|---|---|---|
| Write path | Find page (random seek) → modify in place | Append to WAL + Memtable (sequential) |
| Write amplification | Low per write, page splits occasional | High during compaction (data rewritten multiple times) |
| Read path | Traverse tree → O(log N) | Check Memtable + multiple SSTables |
| Read amplification | Low — O(log N) always | Higher — multiple SSTables (mitigated by Bloom filters) |
| Space amplification | Low — data stored once | Higher — same data in multiple SSTables before compaction |
| Random write perf | Limited by disk seek time | Eliminated — all writes sequential |
| Update model | In-place modification | Append new version, old version cleaned at compaction |
| Delete model | Mark page slot free | Tombstone — cleaned at compaction |
| Background work | None (page splits are synchronous) | Compaction (CPU + I/O intensive) |
| Predictable latency | Yes — O(log N) always | Read latency varies (depends on compaction state) |
| Used by | PostgreSQL, MySQL, SQLite, Oracle | Cassandra, RocksDB, LevelDB, HBase, Kafka log |

---

## 7. RocksDB — LSM-Tree Everywhere

Facebook open-sourced RocksDB (a production-hardened LevelDB fork). It's now embedded in:

- **Kafka** (log segment storage in newer versions)
- **TiKV** (distributed KV store powering TiDB)
- **CockroachDB** (storage layer)
- **MyRocks** (MySQL storage engine replacing InnoDB for write-heavy workloads at Facebook)
- **Flink** (state backend)
- **Yugabyte**

RocksDB shows LSM-Tree is not just a NoSQL thing — it's a general write-optimised storage engine used even inside SQL databases.

---

## 8. ShopSphere Mapping

```
Service           Storage Engine    Reason
──────────────────────────────────────────────────────────────
Order Service     PostgreSQL        B-Tree (InnoDB-like heap)
(PostgreSQL)      — read patterns   Checkout reads need O(log N) guarantee
                  dominate writes   Complex queries, joins, ACID

Analytics Svc     Cassandra         LSM-Tree
(Cassandra)       — write dominant  Millions of events/sec
                  time-series       TWCS compaction for time-windowed data

Search Service    Elasticsearch     LSM-Tree (Lucene segments)
(Elasticsearch)   — write on index  Inverted index built via LSM-style
                  build, read on    segment merging
                  query

Cart/Session      Redis             In-memory (no LSM/B-Tree —
(Redis)           — pure memory     persistence via AOF = LSM-like append)
```

**Why Cassandra's LSM-Tree is perfect for ShopSphere analytics:**

```
ShopSphere event stream:
  - 50,000 page views/minute
  - 10,000 add-to-cart events/minute
  - 2,000 order placements/minute
  Total: ~62,000 writes/minute = ~1,033 writes/second

Cassandra LSM path:
  Each write: WAL append + Memtable insert ≈ 10 microseconds
  At 1,033 writes/sec: trivial — Memtable barely moves
  
PostgreSQL B-Tree path at same rate:
  Each write: find leaf page (random read) + write (random write)
  At 1,033 writes/sec with random I/O: 
  → 1,033 random writes/sec on NVMe = manageable but burning write IOPS
  → At 100,000 writes/sec: PostgreSQL primary is bottlenecked
  → Cassandra at same rate: still in Memtable territory
```

---

## 9. Interview Q&A

**Q: Why does Cassandra favour writes over MySQL?**
A: Cassandra uses an LSM-Tree storage engine. Every write goes to an in-memory Memtable (plus a sequential WAL append for durability) and immediately returns success — no disk seek, no page lookup, no random write. MySQL InnoDB uses a B-Tree where every write must find the correct page on disk (random seek), modify it in-place, and write it back (random write). The random I/O bottleneck of B-Tree caps write throughput; LSM-Tree's sequential-only writes scale to millions of writes/second.

**Q: If LSM-Trees are so fast for writes, why doesn't PostgreSQL use one?**
A: PostgreSQL is designed for complex read-heavy OLTP workloads — arbitrary queries, joins, aggregations, secondary indexes. B-Tree guarantees O(log N) for any key lookup with predictable latency. LSM-Tree's read path has variable latency — it must check multiple SSTables, compaction state affects performance, and Bloom filters only probabilistically eliminate SSTables. For a query engine serving complex SQL, predictable read performance is more valuable than maximum write throughput. PostgreSQL also supports SSDs where B-Tree random writes are fast enough for most workloads.

**Q: What is write amplification in LSM-Trees?**
A: Write amplification is the ratio of bytes written to disk vs bytes of actual data written. In LSM-Tree, data is written multiple times: first to WAL, then as an SSTable when Memtable flushes, then rewritten during compaction (L0→L1→L2...). Each compaction rewrites the data. In levelled compaction, write amplification can reach 10–30x — 1 byte of user data causes 10–30 bytes of disk writes total. This is the LSM trade-off: eliminate random writes (great for SSDs and HDDs) but pay with total volume of bytes written (SSD wear, I/O bandwidth).

**Q: What is a tombstone and why can it cause read performance problems in Cassandra?**
A: SSTables are immutable — you can't delete from them. Deletes write a tombstone marker instead. During reads, Cassandra must scan through tombstones to find live data — if many rows were deleted, a query might process thousands of tombstones before finding (or not finding) actual data. This is called tombstone overhead. Solutions: use TTLs instead of explicit deletes (Cassandra handles TTL expiry natively, more efficiently than tombstones), run compaction aggressively to remove tombstones, and set `gc_grace_seconds` appropriately so tombstones are collected after sufficient time.

**Q: Explain the read path in an LSM-Tree and how Bloom filters help.**
A: A read must check: (1) the current Memtable (in memory — fast), (2) each SSTable on disk from newest to oldest. Without optimisation, N SSTables means N disk reads. Bloom filters are probabilistic data structures stored per SSTable that answer "is this key in this file?" in O(1) with no disk I/O. A false positive (Bloom says yes, key isn't there) causes an unnecessary disk read — acceptable. A false negative is impossible — if Bloom says no, the key is guaranteed absent. This reduces disk reads from N to the small number of SSTables that actually contain the key.

---

## Stage 4 Complete ✅

That's all 12 topics of the **Databases Deep Dive**:

1. ✅ RDBMS Internals — B-Tree, query planning, ACID
2. ✅ Indexing — primary, secondary, composite, covered, over-indexing
3. ✅ RDBMS Horizontal Scaling — why hard, replicas, pooling, partitioning
4. ✅ Read Replicas — WAL internals, sync/async, lag handling, failover
5. ✅ Sharding — shard keys, hot shards, cross-shard ops, resharding
6. ✅ Consistent Hashing — ring, virtual nodes, minimal reshuffling
7. ✅ NoSQL Categories — Redis, MongoDB, Cassandra, Neo4j
8. ✅ Polyglot Persistence — right DB per access pattern
9. ✅ Connection Pooling — HikariCP internals, PgBouncer, sizing, failure modes
10. ✅ ACID vs BASE — CAP, consistency models, when to use which
11. ✅ Schema Evolution — expand-contract, safe migrations, Flyway
12. ✅ LSM-Tree vs B-Tree — storage engine internals

**Drop Stage 5 topics whenever you're ready.**
