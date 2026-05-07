
## Topic 1: RDBMS Internals — B-Tree Indexes, Query Planning, ACID

---

## 1. B-Tree Indexes — How They Actually Work

### The Problem Without an Index

Imagine your `orders` table has 10 million rows. You run:

```sql
SELECT * FROM orders WHERE user_id = 42;
```

Without an index, PostgreSQL does a **full table scan** — reads every single row, one by one. At 10M rows, that's catastrophic.

---

### What a B-Tree Is

A **B-Tree (Balanced Tree)** is a self-balancing tree where:
- Every node has multiple keys and multiple children
- All leaf nodes are at the **same depth** (that's the "balanced" part)
- Leaf nodes are **linked together** as a doubly-linked list

```
                    [30 | 70]
                  /     |      \
          [10|20]    [40|50|60]   [80|90]
          /  |  \       ...         ...
      leaf  leaf  leaf
```

Each leaf node stores: `(indexed_value → page_pointer)` — a pointer to the actual row on disk.

### How a Lookup Works

Query: `WHERE user_id = 42`

1. Start at root: is 42 < 30? No. Is 42 < 70? Yes → go to middle child
2. Middle child `[40|50|60]`: 42 < 40? No... actually 42 < 40 is false, so follow left pointer of 40
3. Reach leaf node containing `42 → page 1847, offset 32`
4. Fetch that page from disk

**Total comparisons: O(log N).** For 10M rows, that's ~23 comparisons instead of 10,000,000.

### Range Queries — Where B-Tree Shines

```sql
WHERE user_id BETWEEN 40 AND 60
```

1. Find `40` in the tree (O(log N))
2. **Walk the linked list** of leaf nodes rightward until you pass 60

This is why B-Trees dominate for range queries — the leaf-level linked list makes sequential access cheap.

### What's Actually Stored in the Index

In PostgreSQL, a B-Tree index on `orders(user_id)` stores:

```
(user_id_value, ctid)
```

`ctid` = physical location of the row = `(page_number, row_offset_within_page)`. The index is a separate file on disk from the table.

---

## 2. Query Planning

When you fire a query, PostgreSQL doesn't just execute it blindly. The **Query Planner** picks the cheapest execution strategy.

### Steps

```
SQL Query
   ↓
Parser → Parse Tree
   ↓
Rewriter → Rewritten Query (views expanded, rules applied)
   ↓
Planner/Optimizer → Execution Plan
   ↓
Executor → Results
```

### What the Planner Considers

**Access methods:**
- **Sequential Scan** — read entire table, page by page. Wins when selectivity is low (e.g., fetch 80% of rows — index would hurt)
- **Index Scan** — use B-Tree to find rows, then fetch heap pages. Wins for high selectivity (e.g., `WHERE id = 5`)
- **Index-Only Scan** — if all needed columns are in the index, skip heap entirely
- **Bitmap Heap Scan** — collect all matching ctids from index first, sort by page, then fetch heap pages in order (reduces random I/O)

**Join strategies:**
- **Nested Loop** — for small tables or indexed lookups
- **Hash Join** — build hash table on smaller side, probe with larger side
- **Merge Join** — both sides must be sorted; great for large sorted datasets

### Statistics Drive the Planner

PostgreSQL maintains `pg_statistics` — histograms and column statistics collected by `ANALYZE`. The planner uses these to estimate row counts. Wrong statistics → bad plan.

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

This shows you the actual plan chosen, estimated vs actual rows, and cost.

---

## 3. ACID Guarantees — The Real Internals

ACID isn't just a checklist — each property has a concrete mechanism behind it.

### Atomicity — All or Nothing

**Mechanism: Write-Ahead Log (WAL)**

Before any page is modified on disk, PostgreSQL writes the change to the WAL (a sequential log file). If the system crashes mid-transaction:
- On restart, PostgreSQL replays the WAL
- Uncommitted transactions are **rolled back** (their WAL entries are ignored)
- Committed transactions are **redone** if their pages weren't flushed yet

```
BEGIN;
  INSERT INTO orders ...   ← WAL entry written
  UPDATE inventory ...     ← WAL entry written
COMMIT;                    ← WAL commit record written → NOW it's permanent
```

If crash happens before COMMIT → both inserts are undone.

### Consistency — Data Always Valid

The database moves from one valid state to another. Enforced by:
- Constraints (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`)
- Triggers
- Application-level invariants

Consistency is **partly the DB's job, partly the app's job**.

### Isolation — Concurrent Transactions Don't Interfere

This is the hardest one. PostgreSQL uses **MVCC (Multi-Version Concurrency Control)**:

- Every row has `xmin` (transaction that created it) and `xmax` (transaction that deleted/updated it)
- When you read a row, PostgreSQL checks: "Is this version visible to my transaction ID?"
- **Writers don't block readers. Readers don't block writers.**

Isolation levels (weakest → strongest):

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | ✅ possible | ✅ | ✅ |
| Read Committed (PG default) | ❌ prevented | ✅ possible | ✅ |
| Repeatable Read | ❌ | ❌ | ✅ possible |
| Serializable | ❌ | ❌ | ❌ |

### Durability — Committed = Survived

**Mechanism: WAL + fsync**

`COMMIT` doesn't return until the WAL record is **fsynced to disk**. Even if the OS crashes after COMMIT returns, the data is safe because WAL is on durable storage and will be replayed on restart.

This is also why `synchronous_commit = off` is dangerous — it lets PostgreSQL return COMMIT before fsync, risking data loss for speed.

---

## 4. ShopSphere Mapping

| Concept | Where it hits ShopSphere |
|---|---|
| B-Tree index | `orders(user_id)`, `products(category_id)`, `order_items(order_id)` — every FK you query by |
| Query planner | Bitmap heap scan used when fetching all orders for a user (moderate selectivity) |
| WAL | Order placement is atomic — inventory deduction + order insert either both succeed or both roll back |
| MVCC | Concurrent users browsing product catalog don't block each other |
| Isolation | `REPEATABLE READ` for checkout flow — price you saw = price you pay |
| Durability | Payment records are fsynced; no `synchronous_commit = off` on payment DB |

---

## 5. Interview Q&A

**Q: Why is a B-Tree better than a hash index for range queries?**
A: Hash index maps key → bucket via hash function. There's no ordering, so `BETWEEN` requires a full scan. B-Tree preserves order and links leaves, so range traversal is O(log N + K) where K is the result size.

**Q: What is an Index-Only Scan and when does it apply?**
A: When every column in the SELECT and WHERE clause is present in the index, PostgreSQL can satisfy the query entirely from the index file without touching the heap. Requires that the visibility map shows all pages as "all-visible" (i.e., no dead tuples that MVCC needs to re-check).

**Q: How does MVCC avoid read-write contention?**
A: Each transaction sees a snapshot of the database as of its start time. Older row versions are kept alive for in-progress transactions. Readers see old versions; writers create new versions. No locks needed for reads.

**Q: What's the cost of `COMMIT` in terms of I/O?**
A: At minimum, one sequential write + fsync to the WAL file. This is why WAL is always on a fast, sequential-write-friendly disk. PostgreSQL batches multiple commits into one fsync via group commit to reduce I/O overhead.

**Q: You have a table with 50M rows and a query that fetches 60% of them. Would you add an index?**
A: No. The planner will correctly choose a sequential scan. An index scan at 60% selectivity means 30M random heap page fetches — far worse than one sequential pass. Indexes help when selectivity is high (small % of rows returned).

---

Ready for **Topic 2: Indexing — primary vs secondary, composite, covered index, and the cost of over-indexing**?
