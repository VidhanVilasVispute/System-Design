# Stage 4 — Databases Deep Dive
## Topic 11: Schema Evolution — Safe Migrations, Backward Compatibility, Rolling Deployments

---

## 1. Why Schema Evolution is Hard in Production

In development: `DROP TABLE`, recreate, done.

In production with live traffic: a schema change is a **dangerous operation** that must be invisible to users and zero-downtime for the application.

The core tension:

```
You want to:          But:
─────────────────────────────────────────────────────
Add a column          Old app instances don't know about it
Rename a column       Old code uses the old name → breaks
Drop a column         Old code still selects it → breaks
Add a NOT NULL col    Existing rows have no value → constraint fails
Add an index          Table lock on small DBs, blocks writes on large ones
Change a column type  Old data might not cast → fails
```

The real constraint: **during a rolling deployment, old and new versions of your service run simultaneously.** A migration that only works with the new code will break the old code still handling requests.

---

## 2. The Expand-Contract Pattern

The only safe pattern for zero-downtime schema changes. Also called **parallel change** or **strangler migration**.

Every breaking schema change is decomposed into three phases deployed independently:

```
Phase 1 — EXPAND:   Add new structure (backward compatible)
Phase 2 — MIGRATE:  Backfill / dual-write / transition
Phase 3 — CONTRACT: Remove old structure (after all code uses new)
```

Between each phase: a full deployment with the system running stable.

---

## 3. Safe Operations — Can Do Without Downtime

These are non-breaking by default:

| Operation | Safe? | Notes |
|---|---|---|
| Add nullable column | ✅ | Old code ignores it, new code uses it |
| Add column with default | ✅ (PG 11+) | PG 11+ stores default in catalog, no table rewrite |
| Add an index `CONCURRENTLY` | ✅ | No table lock — covered below |
| Add a new table | ✅ | Old code doesn't know it exists |
| Widen a column (VARCHAR(50)→VARCHAR(100)) | ✅ | No data change required |
| Drop an index | ✅ | Doesn't affect data |
| Add a nullable FK | ✅ | No constraint on existing rows |

---

## 4. Unsafe Operations — Need the Expand-Contract Pattern

### Case 1: Renaming a Column

Naive: `ALTER TABLE orders RENAME COLUMN user_id TO customer_id;`

Old code still does `SELECT user_id FROM orders` → column not found → crash.

**Safe approach:**

```
Phase 1 — EXPAND:
  Add new column:
  ALTER TABLE orders ADD COLUMN customer_id UUID;
  
  Deploy new code that writes to BOTH user_id and customer_id.
  Reads still from user_id.

Phase 2 — MIGRATE:
  Backfill: UPDATE orders SET customer_id = user_id WHERE customer_id IS NULL;
  
  Deploy new code that reads from customer_id, writes to both.
  Old code still reads user_id (which is still there).

Phase 3 — CONTRACT:
  Verify no old code instances running.
  Deploy code that only uses customer_id.
  Then: ALTER TABLE orders DROP COLUMN user_id;
```

Timeline: 3 separate deployments, days or weeks apart. Tedious — which is exactly why you design schemas carefully upfront.

---

### Case 2: Adding a NOT NULL Column

Naive: `ALTER TABLE orders ADD COLUMN region VARCHAR(10) NOT NULL;`

Fails immediately — existing rows have no value for `region`. Constraint violation.

**Safe approach:**

```
Phase 1 — EXPAND:
  Add as nullable with a default:
  ALTER TABLE orders ADD COLUMN region VARCHAR(10) DEFAULT 'IN';
  
  In PG 11+: this is instant — the default is stored in the catalog,
  not written to every row. ✅

Phase 2 — MIGRATE (if needed):
  If you need specific values per row (not just a blanket default):
  UPDATE orders SET region = derive_region(user_id) WHERE region IS NULL;
  Do this in batches to avoid lock escalation.

Phase 3 — CONTRACT:
  Once all rows have values and all code provides the value on insert:
  ALTER TABLE orders ALTER COLUMN region SET NOT NULL;
```

---

### Case 3: Adding a Large Index

`CREATE INDEX ON orders(user_id);` on a 50M row table.

**Standard index creation takes an exclusive lock** on the table — no reads or writes during index build. On a 50M row table: minutes. Unacceptable in production.

**Safe approach:**

```sql
-- CONCURRENTLY: builds index without locking the table
-- Reads and writes continue normally during build
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

`CONCURRENTLY` takes longer (two passes over the table instead of one) and uses more resources, but the table stays fully accessible throughout.

**Caveats:**
- Cannot run inside a transaction block
- If it fails midway, leaves an invalid index — must drop and retry
- Check for invalid indexes: `SELECT * FROM pg_indexes WHERE indexname = 'idx_orders_user_id'` — look for `pg_index.indisvalid = false`

---

### Case 4: Changing a Column Type

`ALTER TABLE orders ALTER COLUMN amount TYPE BIGINT;`

PostgreSQL rewrites the entire table (a full table scan + new heap file). On large tables: hours, table locked throughout.

**Safe approach:**

```
Phase 1 — EXPAND:
  Add new column with new type:
  ALTER TABLE orders ADD COLUMN amount_new BIGINT;

Phase 2 — MIGRATE:
  Backfill in batches:
  UPDATE orders SET amount_new = amount::BIGINT 
  WHERE id BETWEEN 0 AND 100000;
  -- repeat in batches
  
  Deploy code that writes to both columns.

Phase 3 — CONTRACT:
  Rename (using rename pattern above) or swap:
  ALTER TABLE orders ALTER COLUMN amount_new SET NOT NULL;
  -- then drop amount, rename amount_new to amount
```

---

### Case 5: Dropping a Column

`ALTER TABLE orders DROP COLUMN legacy_field;`

Old code still does `SELECT legacy_field` → error.

**Safe approach:**

```
Phase 1 — CONTRACT prep:
  Remove all references to legacy_field from code.
  Deploy.
  Verify no queries reference it (check slow query logs, application logs).

Phase 2 — CONTRACT:
  ALTER TABLE orders DROP COLUMN legacy_field;
```

**PostgreSQL shortcut:** In PG, `DROP COLUMN` is actually fast — it marks the column as dropped in the catalog (`pg_attribute.attisdropped = true`) without rewriting the table. The column data stays on disk but is invisible to queries. Table rewrite only happens during the next `VACUUM FULL` or `CLUSTER`. So the drop itself is safe to run; just make sure no code references it.

---

## 5. Batched Backfills — How to Migrate Data Without Killing Production

Backfill: `UPDATE orders SET customer_id = user_id WHERE customer_id IS NULL;`

On 50M rows: this single statement holds locks on every row it touches for its entire duration. The DB is effectively unusable during the update.

**Right approach — batch with delay:**

```sql
DO $$
DECLARE
  batch_size INT := 10000;
  last_id    UUID := '00000000-0000-0000-0000-000000000000';
  rows_updated INT;
BEGIN
  LOOP
    UPDATE orders
    SET customer_id = user_id
    WHERE id > last_id
      AND customer_id IS NULL
    ORDER BY id
    LIMIT batch_size
    RETURNING id INTO last_id;

    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;

    PERFORM pg_sleep(0.05);  -- 50ms pause between batches
  END LOOP;
END $$;
```

Each batch:
- Locks only 10,000 rows
- Commits immediately (short lock window)
- 50ms sleep gives normal traffic time to proceed

Total time: longer, but zero impact on production throughput.

---

## 6. Flyway / Liquibase — Migration Management

You never run `ALTER TABLE` by hand in production. You use a migration tool that:
- Tracks which migrations have run (via a version table)
- Runs migrations in order at startup or deploy time
- Prevents the same migration from running twice
- Fails fast if the DB state doesn't match expectations

### Flyway (what ShopSphere uses with Spring Boot)

```
src/main/resources/db/migration/
  V1__create_users_table.sql
  V2__create_orders_table.sql
  V3__add_customer_id_column.sql       ← EXPAND phase
  V4__backfill_customer_id.sql         ← MIGRATE phase (batched)
  V5__add_not_null_customer_id.sql     ← CONTRACT phase
  V6__drop_user_id_column.sql          ← CONTRACT phase
```

Flyway runs outstanding migrations at Spring Boot startup. In a rolling deployment where 3 instances restart one at a time:

```
Instance 1 restarts → runs V3 (ADD COLUMN) → starts serving traffic
Instance 2 restarts → V3 already ran → starts serving traffic
Instance 3 restarts → V3 already ran → starts serving traffic
```

The migration runs exactly once. Old instances (still running V2 code) see the new column and ignore it (nullable, so no issue).

**Critical rule: migrations must be backward compatible.** The migration in a deployment must not break the old code still running. That's the entire point of expand-contract.

### Flyway Configuration in Spring Boot

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    out-of-order: false         # enforce strict version ordering
    validate-on-migrate: true   # fail if checksum of applied migration changed
```

**Never edit a migration that has already been applied.** Flyway checksums every migration. If you change an applied migration file, Flyway throws `FlywayValidationException` on startup and refuses to start. Always add a new migration file.

---

## 7. Zero-Downtime Deployment Checklist

```
Before writing migration:
  □ Is it backward compatible? (old code + new schema = no crash?)
  □ Is it forward compatible? (new code + old schema = no crash?)
  □ Does it need expand-contract? (rename, type change, not-null add)
  □ Is it a large table? (use CONCURRENTLY for indexes, batch for updates)

Migration file:
  □ New V{N}__description.sql file (never edit existing)
  □ Additive only in EXPAND phase
  □ Batched backfill in MIGRATE phase
  □ Destructive only in CONTRACT phase (after all old code is gone)

Deployment order:
  □ Run EXPAND migration → deploy new code (writes both old + new)
  □ Run MIGRATE (backfill data)
  □ Confirm all instances on new code
  □ Run CONTRACT migration → deploy final code (uses only new)
```

---

## 8. ShopSphere Example — Adding `region` to Orders

**Scenario:** Business requirement — add `region` to orders for logistics routing.

```sql
-- V7__add_region_to_orders_expand.sql  (EXPAND — deploy with v2 code)
ALTER TABLE orders ADD COLUMN region VARCHAR(10) DEFAULT 'IN';

-- No code change needed yet — default handles existing rows
-- New orders: old code doesn't set region (gets default 'IN')
```

```sql
-- V8__backfill_orders_region.sql  (MIGRATE — run as background job or migration)
-- Batch update: set correct region based on user's address
UPDATE orders o
SET region = (
  SELECT a.region FROM addresses a 
  WHERE a.user_id = o.user_id 
  AND a.is_default = true 
  LIMIT 1
)
WHERE o.region = 'IN'  -- only rows with default value
AND o.created_at > '2024-01-01';  -- scope to avoid locking old archived rows
```

```java
// New Order Service code (v2) — sets region explicitly
Order order = Order.builder()
    .userId(userId)
    .region(userService.getDefaultRegion(userId))  // new field
    .items(items)
    .build();
// Old code (v1) still running → doesn't set region → gets DEFAULT 'IN' → OK
```

```sql
-- V9__add_not_null_region.sql  (CONTRACT — after all v1 instances gone)
-- Verify first: SELECT COUNT(*) FROM orders WHERE region IS NULL; -- must be 0
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;
ALTER TABLE orders ALTER COLUMN region DROP DEFAULT;  -- force explicit value going forward
```

---

## 9. Interview Q&A

**Q: What is a zero-downtime migration and how do you achieve it?**
A: A zero-downtime migration is a schema change that doesn't require taking the application offline or blocking normal traffic. Achieved via the expand-contract pattern: first add new structure that's backward compatible (EXPAND), then migrate existing data in batches (MIGRATE), then remove old structure after all code has been updated (CONTRACT). Each phase is a separate deployment. The key constraint: during a rolling deployment, old and new code run simultaneously — the migration must not break either version.

**Q: Why can't you just run `ALTER TABLE orders ADD COLUMN region VARCHAR NOT NULL` directly?**
A: Two problems. First, existing rows have no value for `region` — the NOT NULL constraint fails immediately on the ALTER. Second, even if you add a DEFAULT, old application code doesn't provide this column on INSERT — those inserts would fail. The safe path: add as nullable with a default (old code inserts succeed, gets default), backfill existing rows, then once all code explicitly provides the value, add the NOT NULL constraint.

**Q: What does `CREATE INDEX CONCURRENTLY` do differently from a normal index creation?**
A: Normal index creation takes an `AccessShareLock` on the table — blocking all writes for the duration (minutes on large tables). `CONCURRENTLY` builds the index in two passes without holding a table lock — the table stays fully readable and writable throughout. It works by scanning the table once to build the index, then scanning again to catch any changes that happened during the first scan. It takes longer and slightly more CPU/I/O, but is the only safe option for production tables that can't tolerate write blocking.

**Q: Your team changed an already-applied Flyway migration file. What happens?**
A: Flyway stores a checksum of every applied migration in its `flyway_schema_history` table. On the next startup, it recomputes the checksum of all migration files and compares against stored values. If they differ, it throws `FlywayValidationException` and refuses to start. This is intentional — it prevents silent schema drift where different environments have different effective schemas. Fix: revert the file change and add a new migration file with the intended change.

**Q: How do you backfill 100 million rows without killing production traffic?**
A: Batch the updates — process 5,000–10,000 rows per transaction with a short sleep between batches (50–100ms). Each batch holds row locks for only the duration of that small transaction, then releases. Total time increases but production queries can proceed between batches. Use a cursor or keyset pagination (WHERE id > last_processed_id) to avoid OFFSET performance degradation. Monitor replication lag during the backfill — heavy write batches can cause replica lag to spike. Slow down batch rate if lag grows beyond threshold.

**Q: What's the difference between a backward compatible and forward compatible migration?**
A: Backward compatible: the new schema works with the old code — old code can still read and write without errors (e.g., adding a nullable column — old code ignores it). Forward compatible: the new code works with the old schema — new code can start before the migration runs (e.g., new code treats a missing column as null and handles gracefully). Both are needed for true zero-downtime rolling deployments where old and new code run simultaneously with one shared database.

---

Ready for **Topic 12: LSM-Tree vs B-Tree — why Cassandra favours writes, why MySQL favours reads, the internals**?
