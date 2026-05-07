# Stage 4 — Databases Deep Dive
## Topic 2: Indexing — Primary, Secondary, Composite, Covered, and the Cost of Over-Indexing

---

## 1. Primary Index

A **primary index** is built on the primary key. In most RDBMS (PostgreSQL, MySQL InnoDB), this is a special index because of how data is physically stored.

### Clustered vs Non-Clustered (Critical Distinction)

**MySQL InnoDB — Clustered Primary Index:**

The table data itself is stored **inside** the B-Tree leaf nodes, ordered by primary key. The leaf node IS the row.

```
B-Tree Leaf (InnoDB):
[ PK=1 | name="Alice" | email="a@x.com" | ... ]
[ PK=2 | name="Bob"   | email="b@x.com" | ... ]
[ PK=3 | ...                                  ]
```

This means:
- `SELECT * WHERE id = 5` → one B-Tree traversal, data is right there
- Inserts with sequential PKs (auto-increment) are fast — append to the end
- Inserts with random PKs (UUIDs) are slow — must insert in the middle, causes **page splits**

**PostgreSQL — Heap-Based (No Clustering by Default):**

Data is stored in a heap (unordered). The primary key index is a regular B-Tree that stores `(pk_value → ctid)`. The rows themselves are wherever they landed on disk.

```
Index Leaf:          Heap Page:
(id=5 → ctid)   →   actual row data scattered on disk
```

You can run `CLUSTER` to physically reorder the heap once, but it doesn't stay sorted on future inserts.

---

## 2. Secondary Index

Any index that is **not** the primary key. Built on columns you frequently query by but aren't the PK.

```sql
-- Secondary index on user_id in orders table
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

### How Secondary Indexes Reference Rows

**PostgreSQL:** Secondary index leaf stores `(indexed_value → ctid)` — direct physical pointer to heap.

**MySQL InnoDB:** Secondary index leaf stores `(indexed_value → primary_key_value)`. To fetch the full row, it does a **second B-Tree lookup** into the clustered index. This is called a **double lookup** or **bookmark lookup**.

```
Secondary Index Lookup (MySQL):
user_id=42 → PK=1007 → [look up PK=1007 in clustered index] → full row
```

This is why in MySQL, keeping your PK small (INT vs UUID) matters — every secondary index carries a copy of the PK in its leaf nodes.

---

## 3. Composite Index

An index on **multiple columns**.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### The Left-Prefix Rule — Most Important Rule in Indexing

A composite index `(A, B, C)` can be used for queries on:
- `A` alone ✅
- `A, B` ✅
- `A, B, C` ✅
- `B` alone ❌ — can't use the index
- `A, C` — partially ✅ (uses A, then scans for C)

**Why?** The B-Tree is sorted first by A, then by B within the same A, then by C within the same B. Without A, there's no starting point.

```
Index (user_id, status):
[user_id=1, status=PENDING]
[user_id=1, status=SHIPPED]
[user_id=2, status=PENDING]
[user_id=2, status=DELIVERED]
...
```

Query: `WHERE user_id = 1 AND status = 'PENDING'` → index locates exact leaf directly ✅  
Query: `WHERE status = 'PENDING'` → no starting point, full index scan ❌

### Column Order Matters

Put the **most selective** column first, **unless** you have an equality filter on one and a range on the other — then put the equality column first.

```sql
-- Good: equality on user_id (high selectivity), range on created_at
CREATE INDEX ON orders(user_id, created_at);

-- Query this serves perfectly:
WHERE user_id = 42 AND created_at > '2024-01-01'
```

If you flip it to `(created_at, user_id)`, the range on `created_at` kills the ability to use `user_id` efficiently.

---

## 4. Covered Index (Index-Only Scan)

A **covered index** (or covering index) is one that contains **all columns** a query needs — so the DB never touches the heap at all.

```sql
-- Query:
SELECT status, created_at FROM orders WHERE user_id = 42;

-- Covered index:
CREATE INDEX idx_orders_covering ON orders(user_id, status, created_at);
```

The index leaf node now contains `(user_id, status, created_at)` — all three columns. The query is satisfied entirely from the index file.

**Why is this a big deal?**

| Step | Normal Index Scan | Index-Only Scan |
|---|---|---|
| 1 | Traverse B-Tree | Traverse B-Tree |
| 2 | Get ctid/PK | ✅ Done — data is here |
| 3 | Random heap fetch | — |

Random heap fetches are the expensive part — each one may require a different disk page. Eliminating them can make a query **5–10x faster** on large tables.

### PostgreSQL Caveat — Visibility Map

In PostgreSQL, even for an index-only scan, the engine must verify row visibility (MVCC). It checks the **visibility map** — a bitmap that tracks which heap pages are "all-visible" (all rows confirmed committed). If the page is all-visible, skip heap. If not, fetch the heap page to check `xmin/xmax`. `VACUUM` is what keeps the visibility map up-to-date — without regular vacuuming, index-only scans degrade to regular index scans.

---

## 5. The Cost of Over-Indexing

This is where most junior engineers go wrong — indexes feel free because reads get faster. They're not free.

### Every Index You Add:

**Slows down writes:**
Every `INSERT`, `UPDATE`, `DELETE` must update every index on that table. A table with 8 indexes means 8 B-Tree modifications per write, each potentially causing page splits and rebalancing.

```
INSERT INTO orders(...) 
→ update clustered/heap
→ update idx_orders_user_id
→ update idx_orders_status
→ update idx_orders_created_at
→ update idx_orders_covering
→ ... (8 total)
```

**Consumes disk space:**
Each index is a separate file on disk. For a 100GB orders table with 6 indexes, you might have 200–300GB of index files.

**Inflates memory pressure:**
PostgreSQL's `shared_buffers` caches both heap pages and index pages. Too many indexes → index pages evict hot table pages → more disk I/O overall.

**Confuses the query planner:**
With many indexes, the planner must evaluate more execution plan candidates. Occasionally it picks a worse index if statistics are stale.

### Practical Rules

| Situation | Action |
|---|---|
| FK columns (`user_id`, `order_id`) | Always index — JOINs and lookups need them |
| Columns in frequent `WHERE` clauses | Index, but check query selectivity first |
| Columns only in `SELECT` | Don't index alone — use covering index instead |
| Low-cardinality columns (`status` with 3 values) | Skip standalone index — use composite |
| Tables with heavy write load | Be conservative — profile before adding |

---

## 6. ShopSphere Mapping

**Order Service — `orders` table:**

```sql
-- Primary
PK: order_id (UUID — be careful of page splits in MySQL)

-- Secondary
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Composite: dashboard query — user's orders filtered by status
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Covering: order listing page only needs these columns
CREATE INDEX idx_orders_listing ON orders(user_id, created_at DESC, status, total_amount);
```

**Product Service — `products` table:**

```sql
-- Composite for browse-by-category with sort
CREATE INDEX idx_products_category_price ON products(category_id, price);

-- Full-text search goes to Elasticsearch, not a DB index
```

**Over-indexing trap in ShopSphere:**
The `order_items` table is written every time an order is placed. If you add 5 indexes to it for analytics queries — write throughput tanks during flash sales. Analytics queries should go to a **read replica** with its own index set, not pollute the write master.

---

## 7. Interview Q&A

**Q: Why does a UUID primary key hurt performance in MySQL?**
A: InnoDB stores rows in PK order (clustered index). UUIDs are random, so every insert lands in a random position in the B-Tree, causing frequent **page splits** — the engine has to split a full leaf page into two and redistribute entries. Auto-increment integers append to the end — no splits, much faster inserts and more compact pages.

**Q: You have a composite index on (A, B). A query filters on B only. Will it use the index?**
A: No, unless the planner decides a full index scan is cheaper than a heap scan — but even then, it's not using the index for its selectivity. The left-prefix rule means B alone can't seek into the tree efficiently. Solution: add a separate index on B, or restructure the composite to (B, A).

**Q: What's the difference between a composite index and a covering index?**
A: These aren't mutually exclusive. A composite index is just an index on multiple columns. A covering index is any index that contains all columns a specific query needs, enabling an index-only scan. A composite index becomes a covering index for a particular query if that query doesn't need any columns outside the index.

**Q: How do you find unused indexes in PostgreSQL?**
A: Query `pg_stat_user_indexes` — it tracks `idx_scan` (how many times the index was used). Indexes with `idx_scan = 0` after sufficient traffic are candidates for removal. Also check `pg_stat_user_tables` for `seq_scan` vs `idx_scan` ratio per table.

**Q: When would you deliberately NOT index a foreign key?**
A: On the "one" side of a one-to-many when you never query from that direction. For example, `orders.payment_method_id` FK to `payment_methods` — if you never do `WHERE payment_method_id = X` on orders, the index is pure write overhead. Index FKs you JOIN or filter on; audit the rest.

---

Ready for **Topic 3: RDBMS Horizontal Scaling — why it's hard, read replicas, connection pooling, and partitioning**?
