# Stage 7 — Distributed Systems
## Topic 13: Write-Ahead Log (WAL)

---

## The Problem — Durability Under Crashes

Your database confirms a write. One millisecond later — power failure.

When the database restarts, is that write still there?

The naive implementation says no:

```
Naive database write path:
──────────────────────────────────────────────────────────────
1. Receive write: UPDATE orders SET status='CONFIRMED' WHERE id=X
2. Find the page in memory (buffer pool)
3. Modify the page in memory
4. Eventually flush to disk

Problem: "Eventually" might be 30 seconds later.
If crash happens between step 3 and step 4:
  → Memory is gone
  → Disk has old value
  → Write is LOST
  → Client was already told "success" ✅ ← LIE
──────────────────────────────────────────────────────────────
```

**The D in ACID — Durability** — means confirmed writes survive crashes. How?

Flush every write to disk immediately? Disks are slow. A magnetic disk does ~200 random writes/sec. An SSD does ~10,000. Neither can keep up with thousands of transactions per second.

**The Write-Ahead Log is the solution.**

---

## The Core Idea

> **Before modifying any data page, write a description of the change to an append-only log on disk. Only THEN modify the actual data.**

```
WAL principle:
  "Log first. Data second. Always."

The log is:
  → Append-only  (sequential writes — fastest disk operation possible)
  → Durable      (fsync'd before confirming to client)
  → Sufficient   (log entry alone is enough to redo the change)
```

Sequential writes are **10–100x faster** than random writes on both magnetic and SSD disks. The WAL exploits this by turning all writes into sequential appends to one file, regardless of where in the database the actual data lives.

---

## WAL Structure

```
WAL file on disk:

Offset  LSN         Content
──────────────────────────────────────────────────────────────────
0       0/01000000  CHECKPOINT (starting point)
1       0/01000028  BEGIN  txn_id=1042
2       0/01000060  UPDATE orders SET status='CONFIRMED' WHERE id=X
                    → old value: 'PLACED'
                    → new value: 'CONFIRMED'
                    → page: heap page 47, offset 128
3       0/010000A0  UPDATE inventory SET stock=9 WHERE product_id=42
                    → old value: 10
                    → new value: 9
                    → page: heap page 12, offset 64
4       0/010000E0  COMMIT txn_id=1042
5       0/01000100  BEGIN  txn_id=1043
...
```

**LSN = Log Sequence Number** — a monotonically increasing identifier for every WAL record. Critically important — we'll see it everywhere.

Each WAL record contains:
```
→ Transaction ID
→ Operation type (INSERT / UPDATE / DELETE / COMMIT / ROLLBACK)
→ Table and page location
→ Old value (for rollback / undo)
→ New value (for redo)
→ LSN of previous record (linked list structure)
```

---

## The Write Path — Step by Step

```
Client sends: UPDATE orders SET status='CONFIRMED' WHERE id=X

Step 1: Database finds the relevant page
        → Check buffer pool (memory) first
        → If not in memory → read from disk into buffer pool

Step 2: WAL record written to WAL buffer (memory)
        "Intent to change: orders page 47, offset 128,
         old=PLACED, new=CONFIRMED, txn=1042"

Step 3: WAL buffer FLUSHED to disk (fsync)
        ← this is the durability guarantee moment
        ← sequential append = fast

Step 4: Data page modified IN MEMORY (buffer pool)
        ← NOT yet on disk

Step 5: "COMMIT" WAL record written + flushed to disk

Step 6: Client told: "COMMIT successful" ✅

Step 7: (Later, async) Modified pages flushed to disk
        ← background writer / checkpoint process
```

```
Timeline:
──────────────────────────────────────────────────────────────
WAL flush  ──▶ [data page in memory]  ──▶ [data page on disk]
    │                                            │
    │◀─── durability guaranteed here             │◀─ performance optimization
    │     (crash here → WAL has full record)     │   (can lag by seconds/minutes)
```

If a crash happens AFTER step 3 but before step 7 — the WAL has the complete change record. On recovery, the database replays it. Durability preserved. ✅

---

## Crash Recovery — ARIES Algorithm

When PostgreSQL restarts after a crash, it runs the **ARIES recovery algorithm** (Analysis, Redo, Undo):

```
Phase 1 — ANALYSIS:
  Read WAL from last checkpoint forward
  Build two tables:
    → Dirty page table: pages modified in memory but not yet on disk
    → Transaction table: which transactions were in-flight at crash

Phase 2 — REDO:
  Replay ALL WAL records from the earliest dirty page LSN
  Even transactions that already committed — re-apply their changes
  (Data pages on disk might be behind WAL — must catch up)
  After redo: disk state matches what it was at crash time ✅

Phase 3 — UNDO:
  Find all transactions that were in-progress at crash (no COMMIT record)
  Apply their UNDO operations in reverse order
  (Roll back incomplete transactions — atomicity restored)
  After undo: only committed transactions visible ✅
```

```
Example crash recovery:

WAL at crash time:
  LSN 100: BEGIN txn=1
  LSN 101: INSERT orders (id=X)          ← txn 1
  LSN 102: COMMIT txn=1                  ← committed ✅
  LSN 103: BEGIN txn=2
  LSN 104: UPDATE inventory stock=9      ← txn 2, in-progress
  [CRASH — no COMMIT for txn 2]

Recovery:
  Analysis: txn 1 committed, txn 2 in-progress
  Redo:     replay LSN 101 (insert), LSN 104 (update) → disk caught up
  Undo:     reverse LSN 104 → stock back to 10 (txn 2 never committed)

Final state: order X exists ✅, stock = 10 ✅ (txn 2 rolled back)
```

---

## PostgreSQL WAL In Depth

PostgreSQL's WAL is stored in `$PGDATA/pg_wal/` as 16MB segment files:

```
/var/lib/postgresql/data/pg_wal/
  000000010000000000000001
  000000010000000000000002
  000000010000000000000003
  ...
```

**Key PostgreSQL WAL settings:**

```ini
# postgresql.conf

# When does WAL flush happen?
synchronous_commit = on      # flush WAL before confirming COMMIT
                             # off = faster but risk losing last ~1s of commits

# WAL compression
wal_compression = on         # compress WAL records — saves I/O

# WAL level — how much detail to log
wal_level = replica          # minimum for streaming replication
wal_level = logical          # needed for logical replication + Debezium

# Checkpoint frequency
checkpoint_timeout = 5min    # force checkpoint every 5 minutes
max_wal_size = 1GB           # trigger checkpoint if WAL grows beyond this
```

**`synchronous_commit = off`** is a common performance optimization:

```
synchronous_commit = on  (default):
  COMMIT → WAL flushed to disk → client notified
  Safe: zero data loss
  Slower: every commit waits for disk fsync

synchronous_commit = off:
  COMMIT → client notified → WAL flushed async (within ~200ms)
  Risk: up to ~200ms of commits lost on crash
  Faster: no disk wait per commit

Good for: non-critical writes (analytics events, session data)
Bad for:  orders, payments, inventory — never turn off for these
```

---

## WAL and Replication — The Connection

PostgreSQL streaming replication is entirely WAL-based:

```
Primary PostgreSQL:
  → Writes data, generates WAL
  → WAL sender process streams WAL to standbys

Standby PostgreSQL:
  → WAL receiver process receives WAL stream
  → WAL relay applies WAL records to its own data files
  → Standby is essentially replaying primary's WAL in real time

                 Primary
                    │
          WAL stream│ (continuous, binary)
                    ▼
    ┌───────────────────────────────┐
    │  Standby 1   │   Standby 2   │
    │  (replica)   │   (replica)   │
    └───────────────────────────────┘
```

**Synchronous replication (quorum write):**

```ini
# Primary waits for standby to confirm WAL receipt before COMMIT
synchronous_standby_names = '1 (standby1, standby2)'
# At least 1 of the listed standbys must confirm
# This is the quorum write we discussed in Topic 7
```

**LSN tracking in replication:**

```sql
-- On primary: check replication lag
SELECT
    client_addr,
    sent_lsn,        -- WAL sent to standby
    write_lsn,       -- WAL written to standby disk
    flush_lsn,       -- WAL flushed (durable) on standby
    replay_lsn,      -- WAL applied to standby data files
    (sent_lsn - replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;
```

If `sent_lsn` >> `replay_lsn` → standby is lagging — reads from standby will be stale.

---

## WAL and Logical Decoding — How Debezium Works

From Topic 12 — Debezium reads the PostgreSQL WAL to implement CDC.

This works via **logical replication** — PostgreSQL decodes WAL records into logical changes (INSERT/UPDATE/DELETE on specific tables):

```
Physical WAL record (binary, internal format):
  "Page 47, offset 128: bytes 0x4F5244455200 → 0x434F4E46..."
  (raw bytes — meaningless to application)

Logical replication output (decoded, human-readable):
  "UPDATE orders SET status='CONFIRMED' WHERE id='uuid-X'"
  (meaningful change event)
```

```
PostgreSQL WAL
      │
      │ logical decoding (pgoutput plugin)
      ▼
Replication Slot ← Debezium connects here like a standby
      │
      │ change events: INSERT/UPDATE/DELETE
      ▼
Debezium connector
      │
      ▼
Kafka topic: "shopsphere.public.outbox" → downstream consumers
```

**Replication slot** holds WAL until Debezium confirms it's consumed:

```sql
-- Create replication slot for Debezium
SELECT pg_create_logical_replication_slot(
    'debezium_slot',    -- slot name
    'pgoutput'          -- output plugin
);

-- Monitor: if Debezium is down, WAL accumulates here!
SELECT slot_name, confirmed_flush_lsn, pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
) AS lag
FROM pg_replication_slots;
```

⚠️ **Critical operational note:** If Debezium stops consuming, PostgreSQL cannot delete old WAL segments (standby might need them). WAL accumulates → disk fills up → database crashes. Monitor replication slot lag in production.

---

## WAL Across Different Systems

The WAL concept appears everywhere — just with different names:

```
System            WAL Equivalent        Notes
──────────────────────────────────────────────────────────────────
PostgreSQL        WAL (pg_wal/)         Segment files, LSN-addressed
MySQL/InnoDB      Redo Log              Circular buffer + binlog
MySQL             Binary Log (binlog)   Logical log for replication
                                        (like PostgreSQL logical WAL)
Kafka             The topic log itself  Kafka IS a distributed WAL
SQLite            WAL mode (WAL file)   Explicit WAL mode option
RocksDB           Write-Ahead Log       Used by CockroachDB, TiKV
MongoDB           Oplog (operations log) Capped collection = WAL
etcd              Raft log              WAL entries = Raft entries
Elasticsearch     Translog              Per-shard WAL
```

**Kafka IS a WAL:**

```
Kafka topic partition = append-only, ordered, durable log
  → Producers append records (WAL appends)
  → Consumers read from offset (WAL replay)
  → Retention period = how long WAL is kept
  → Log compaction = checkpoint + cleanup
  → Replication = WAL streaming to replicas

Kafka doesn't just USE the WAL idea — Kafka IS a distributed WAL
made accessible as a first-class messaging system.
```

---

## Checkpoints — Bounding Recovery Time

If the WAL grows forever, crash recovery would require replaying years of logs.

**Checkpoints** periodically flush all dirty pages to disk and record the checkpoint LSN:

```
Between checkpoints:
  WAL grows: record every change
  Data pages: modified in memory, not yet on disk (dirty)

At checkpoint:
  1. All dirty pages flushed to disk
  2. Checkpoint record written to WAL:
     "As of LSN 0/5A000000, all pages are on disk"
  3. WAL segments before checkpoint can now be deleted

On crash recovery:
  Only need to replay WAL from LAST CHECKPOINT forward
  Everything before checkpoint is already safely on disk
```

```
WAL timeline:
─────────────────────────────────────────────────────────────────
[checkpoint A]──────────[checkpoint B]──────────[CRASH]
      │                       │                    │
      │                       │◀── replay this ────┘
      │◀─ can delete ─────────┘    (only this segment needed)
```

```
Recovery time ∝ distance between last checkpoint and crash
Checkpoint every 5 min → max 5 min of WAL to replay
Checkpoint every 1 min → max 1 min of WAL to replay (faster recovery)
                          but more I/O during normal operation
```

---

## WAL in ShopSphere — Operational Checklist

```
PostgreSQL WAL settings per service:
──────────────────────────────────────────────────────────────────
order-service DB:
  synchronous_commit = on       ← never lose orders
  wal_level = logical           ← Debezium needs logical decoding
  max_wal_senders = 5           ← replication + Debezium slots

payment-service DB:
  synchronous_commit = on       ← financial data, never off
  synchronous_standby_names = '1 (standby1)'  ← quorum write

inventory-service DB:
  synchronous_commit = on       ← stock accuracy critical

notification-service DB:
  synchronous_commit = off      ← losing a notification log is acceptable
                                   10x write throughput improvement

analytics DB:
  synchronous_commit = off      ← high-volume event writes, best-effort
──────────────────────────────────────────────────────────────────
```

**Monitoring WAL in production:**

```sql
-- WAL generation rate (how fast WAL is being written)
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')
) AS total_wal_generated;

-- Check for replication lag
SELECT
    application_name,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag
FROM pg_stat_replication;

-- WAL files on disk
SELECT count(*), pg_size_pretty(sum(size))
FROM pg_ls_waldir();
```

---

## Everything Connects Here

The WAL is the common thread through all the patterns we've covered:

```
Topic 10 — Event Sourcing:
  The event store IS a WAL for your domain.
  Append-only, ordered, replayable — same principles.

Topic 12 — Outbox Pattern (Debezium):
  Debezium reads PostgreSQL WAL to detect outbox inserts.
  WAL is the reliable channel that makes CDC possible.

Topic 11 — CQRS Projections:
  Projections are rebuilt by replaying the event log.
  The event log is the same concept as WAL.

Topic 7 — Quorum Writes:
  PostgreSQL synchronous replication uses WAL streaming.
  Quorum = "WAL confirmed on N standbys."

Topic 5 — Raft Consensus:
  Raft log = WAL for distributed state.
  Leader appends to its Raft log, streams to followers.
  Followers apply entries = WAL replay.

Topic 14 — The Power of the Log (next):
  All of these are the same idea at different scales.
  The log is the universal primitive.
```

---

## Interview Angles 🎯

**Q: What is the Write-Ahead Log and why does it exist?**
> The WAL is an append-only log where databases record every intended change before applying it to actual data pages. It exists because sequential disk writes (appending to a log) are far faster than random writes (updating data pages scattered across disk). By writing to the WAL first and flushing it to disk, the database achieves durability cheaply — confirmed writes survive crashes because the WAL record can always redo the change on recovery.

**Q: Walk me through crash recovery using the WAL.**
> PostgreSQL uses the ARIES algorithm: first Analysis — read WAL from last checkpoint to identify which transactions committed and which were in-flight. Then Redo — replay all WAL records forward from the earliest dirty page, catching disk state up to the crash point. Then Undo — reverse all in-progress transactions that never committed, restoring atomicity. After these three phases, the database is consistent.

**Q: How does PostgreSQL streaming replication work?**
> The primary generates WAL for every write. A WAL sender process on the primary streams these WAL records to standby nodes continuously. Each standby has a WAL receiver that writes the stream to its own WAL, and a recovery process that applies it to the standby's data files. The standby is essentially replaying the primary's WAL in real time — it's crash recovery, running continuously.

**Q: How does Debezium connect to PostgreSQL's WAL?**
> Debezium uses PostgreSQL's logical replication feature, connecting as a replication client via a replication slot. PostgreSQL decodes WAL records into logical change events (human-readable INSERT/UPDATE/DELETE) using the pgoutput plugin. Debezium reads these decoded events and publishes them to Kafka. The replication slot ensures no WAL is discarded until Debezium confirms consumption — which means unmonitored slot lag can fill disk.

**Q: Why is Kafka fundamentally a WAL?**
> Kafka topics are append-only, ordered, durable logs — exactly what a WAL is. Producers append records (like WAL writes), consumers read from offsets (like WAL replay), retention manages log cleanup (like WAL segment deletion after checkpoints), and replication streams the log to brokers (like WAL streaming to standbys). The concepts are identical — Kafka just exposes the WAL as a first-class distributed messaging primitive.

---

Say **next** for **Topic 14: The Power of the Log** — the final topic, tying everything together: why the log is the universal primitive underlying Kafka, databases, consensus, and event-driven systems 🚀
