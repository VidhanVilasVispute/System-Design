# Topic 5 — Redundancy and Replication

## Setting the Stage

Topic 4 told you *what* to make redundant. Topic 5 answers *how* redundancy actually works under the hood — and the critical architectural choice that determines your system's behavior during failure.

```
Redundancy = having more than one copy of a component
Replication = the mechanism that keeps those copies in sync

You can't have useful redundancy without replication.
A second DB node that diverges from the first is not a backup — it's a liability.
```

---

## The Core Choice — Active-Active vs Active-Passive

This is the most important decision in redundancy design.

### Active-Passive (Primary-Standby)

```
NORMAL STATE:
                        ┌─────────────────┐
  All Traffic ─────────▶│   Primary (A)   │  ← serving all requests
                        │   ACTIVE        │
                        └─────────────────┘
                                │
                           replication
                                │
                                ▼
                        ┌─────────────────┐
                        │   Standby (B)   │  ← exists, synced, but SILENT
                        │   PASSIVE       │    takes zero traffic
                        └─────────────────┘

FAILURE STATE (A dies):
                        ┌─────────────────┐
                        │   Primary (A)   │  ✗ DEAD
                        └─────────────────┘

  Failover trigger (auto or manual)
                                │
                                ▼
                        ┌─────────────────┐
  All Traffic ─────────▶│   Standby (B)   │  ← promoted to primary
                        │   NOW ACTIVE    │    serving all requests
                        └─────────────────┘

  Recovery time: seconds (automated) to minutes (manual)
  Standby capacity during normal ops: WASTED
```

### Active-Active

```
NORMAL STATE:
                    ┌─────────────────┐
         ┌─────────▶│    Node A       │  ← serving ~50% traffic
         │          │    ACTIVE       │
Traffic ─┤          └─────────────────┘
         │                │  ↕ sync
         │          ┌─────────────────┐
         └─────────▶│    Node B       │  ← serving ~50% traffic
                    │    ACTIVE       │
                    └─────────────────┘

FAILURE STATE (A dies):
                    ┌─────────────────┐
                    │    Node A       │  ✗ DEAD
                    └─────────────────┘

  Load balancer detects failure, removes A from rotation

                    ┌─────────────────┐
  All Traffic ─────▶│    Node B       │  ← absorbs 100% traffic
                    │    ACTIVE       │    (must have capacity headroom)
                    └─────────────────┘

  Recovery time: near-zero (no promotion needed)
  Capacity during failure: reduced but functional (if headroom planned)
```

---

## Deep Comparison

```
┌──────────────────────┬───────────────────────────┬──────────────────────────┐
│                      │   Active-Passive           │   Active-Active          │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Traffic during       │ 100% → Primary only        │ Split across all nodes   │
│ normal ops           │                            │                          │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Failover time        │ Seconds–minutes            │ Near-zero (LB reroute)   │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Resource utilization │ Standby is idle (wasted)  │ All nodes working        │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Complexity           │ Low — clear primary        │ High — conflicts,        │
│                      │                            │ distributed writes       │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Data conflicts       │ None — one writer          │ Possible — two nodes     │
│                      │                            │ write same record        │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ Best for             │ Databases, stateful        │ Stateless services,      │
│                      │ components                 │ caches, app tier         │
├──────────────────────┼───────────────────────────┼──────────────────────────┤
│ ShopSphere example   │ PostgreSQL Primary +       │ order-service ×3,        │
│                      │ RDS standby                │ API Gateway ×2,          │
│                      │                            │ Redis Cluster            │
└──────────────────────┴───────────────────────────┴──────────────────────────┘
```

---

## Replication Deep Dive

Redundancy is useless without the data being in sync. Let's go through exactly how replication works for each component in ShopSphere.

### PostgreSQL Replication — WAL Streaming

```
WAL = Write-Ahead Log
  Every change to PostgreSQL is first written to the WAL
  (an append-only log on disk) BEFORE the actual data page is modified.

  Purpose of WAL:
    1. Crash recovery (replay WAL on restart)
    2. Replication (stream WAL to replicas)
```

```
Replication Flow:

  ┌─────────────────────────────────────────────────────────┐
  │                PostgreSQL Primary                       │
  │                                                         │
  │  INSERT INTO orders (...)                               │
  │       │                                                 │
  │       ▼                                                 │
  │  WAL Record: { LSN: 0/1A2B3C, op: INSERT, data: ... }  │
  │       │                                                 │
  │       ├──▶ Apply to data pages (actual table)           │
  │       │                                                 │
  │       └──▶ WAL Sender Process ──────────────────────▶   │
  └─────────────────────────────────────────────────────────┘
                                          │
                               TCP streaming (port 5432)
                                          │
                                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │                PostgreSQL Replica                       │
  │                                                         │
  │  WAL Receiver Process ◀──────────────────────────────── │
  │       │                                                 │
  │       ▼                                                 │
  │  Apply WAL record → replica data pages now identical    │
  └─────────────────────────────────────────────────────────┘
```

**Synchronous vs Asynchronous Replication:**
```
Asynchronous (default):
  Primary: write WAL → ACK to client ✅ → stream to replica (background)
  
  Pro: Low latency writes (client doesn't wait for replica)
  Con: If primary crashes before replica receives WAL → DATA LOSS
       (replication lag = potential data loss window)

Synchronous:
  Primary: write WAL → stream to replica → wait for replica ACK
           → only then ACK to client ✅
  
  Pro: Zero data loss — replica always has everything primary has
  Con: Write latency = primary latency + network round trip to replica
       Replica going down = primary write latency spikes or blocks

postgresql.conf:
  synchronous_commit = on     ← synchronous (safe, slower)
  synchronous_commit = off    ← async (fast, small loss window)
  synchronous_commit = remote_write ← middle ground
```

**LSN — Log Sequence Number:**
```
Every WAL record has an LSN (monotonically increasing offset).
Replica reports its current LSN to primary.
Primary knows exactly how far behind each replica is.

pg_stat_replication view:
  SELECT client_addr,
         write_lag,    ← time for WAL to reach replica disk
         flush_lag,    ← time for WAL to be flushed to replica disk
         replay_lag    ← time for WAL to be applied to replica data
  FROM pg_stat_replication;

  write_lag: 0ms   flush_lag: 1ms   replay_lag: 5ms  ← healthy
  replay_lag: 500ms                                   ← replica falling behind
```

---

### Redis Replication

```
Redis uses asynchronous master-replica replication.

Initial sync (replica connects for first time):
  Master: BGSAVE → creates RDB snapshot
  Master: streams RDB to replica
  Replica: loads RDB → now in sync
  Master: streams buffered commands that happened during BGSAVE
  → Replica fully caught up

Ongoing replication:
  Every write command on master →
  replicated to replica as command stream (not WAL)

  Master: SET product:5:price 999
  Replica receives: SET product:5:price 999
  Replica applies → identical state

Replica is READ-ONLY by default:
  All writes → master
  All reads → can go to replicas (read scaling)
```

```
Redis Sentinel — Automated Failover:

  3 Sentinel processes (must be odd number for quorum):

  Normal:
    Sentinel-1 ─┐
    Sentinel-2 ─┼──▶ All monitoring Master
    Sentinel-3 ─┘

  Master dies:
    Step 1: Sentinels detect SDOWN (subjectively down) — one sentinel
    Step 2: Sentinels gossip → agree on ODOWN (objectively down)
            Quorum = majority (2 of 3 here)
    Step 3: Sentinels elect a leader among themselves
    Step 4: Leader picks best replica (least replication lag)
    Step 5: Sends SLAVEOF NO ONE to chosen replica → promotes it
    Step 6: Updates other replicas to follow new master
    Step 7: Notifies clients via pub/sub → clients reconnect

  Total time: 10–30 seconds typically
```

---

### Kafka Replication — ISR Model

```
Topic: order-events
Partitions: 3, Replication Factor: 3, 3 Brokers

Partition 0 assignment:
  Leader:   Broker-1  ← all reads AND writes for partition 0
  Follower: Broker-2  ← synced replica
  Follower: Broker-3  ← synced replica

ISR = In-Sync Replicas
  = set of replicas caught up with the leader
  Initially: ISR = {Broker-1, Broker-2, Broker-3}
```

```
Write flow with acks=all:

  Producer
    │  produce: order-placed {orderId: 99}
    ▼
  Broker-1 (Leader, Partition 0)
    │  writes to local log
    ├──▶ Broker-2 fetches and acknowledges
    └──▶ Broker-3 fetches and acknowledges
    │  all ISR members acked
    ▼
  ACK to Producer ✅  ← only after ALL ISR members confirm

  min.insync.replicas = 2:
    At least 2 replicas must be in ISR for writes to succeed.
    If only 1 replica alive → writes BLOCKED (safety over availability)
```

```
Broker-1 dies:

  Before death: ISR = {Broker-1, Broker-2, Broker-3}
  
  Controller (KRaft in modern Kafka):
    Detects Broker-1 offline
    Elects new leader from ISR:
      Broker-2 becomes leader for Partition 0
    Updates metadata → all producers/consumers redirect to Broker-2
    ISR temporarily: {Broker-2, Broker-3}
    
  No data loss: acks=all guaranteed Broker-2 had all records
  Broker-1 recovers → rejoins as follower → ISR restored
```

---

## What Gets Replicated and Why

```
Component          What's replicated              Why
──────────────────────────────────────────────────────────────────────
PostgreSQL         WAL records (every change)     Exact state sync,
                                                  point-in-time recovery
Redis              Command stream                 Lightweight, fast
Kafka              Log segments (partition data)  Message durability
Elasticsearch      Shard data                     Search index HA
etcd (K8s)         Raft log entries               Cluster state consensus

What's NOT replicated:
  In-flight requests          → handled by LB retry
  Local JVM heap              → stateless services, nothing to replicate
  Local file system of pods   → use PersistentVolumes or object storage
```

---

## The Split-Brain Problem

This is the most dangerous failure mode in active-active or replicated systems.

```
Scenario: Network partition between two nodes

  ┌──────────────┐         ✗ network cut ✗        ┌──────────────┐
  │   Node A     │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │   Node B     │
  │  (thinks     │                                 │  (thinks     │
  │  B is dead)  │                                 │  A is dead)  │
  └──────────────┘                                 └──────────────┘
        │                                                 │
        ▼                                                 ▼
  Promotes itself                                  Promotes itself
  to primary                                       to primary
        │                                                 │
        ▼                                                 ▼
  Accepts writes                                   Accepts writes

  Network recovers:
    Node A has writes X, Y, Z
    Node B has writes X, P, Q
    ← CONFLICT. Which is truth? Data permanently diverged.
```

```
How systems prevent split-brain:

PostgreSQL + Patroni:
  Uses etcd/ZooKeeper as arbiter (quorum-based)
  Node can only become primary if it gets majority vote from etcd
  Network partition → isolated node can't get quorum → stays standby
  Only one node has quorum → only one primary at a time

Kafka (KRaft):
  Controller quorum (Raft consensus algorithm)
  Leader election requires majority of controllers to agree
  Minority partition can't elect a leader → reads-only, no writes

Redis Sentinel:
  Requires quorum (majority of sentinels) to promote
  Minority partition of sentinels can't promote
  → isolated master eventually steps down (loses sentinel contact)
```

---

## Replication Topologies

### Single Primary → Multiple Replicas (most common)

```
          ┌──────────────┐
          │   Primary    │
          └──────────────┘
           /      |      \
          /       |       \
    ┌─────┐   ┌─────┐   ┌─────┐
    │ R-1 │   │ R-2 │   │ R-3 │
    └─────┘   └─────┘   └─────┘

  + Simple, clear primary
  + Multiple read replicas scale reads
  - Primary is write bottleneck
  - Primary dies → must elect new one
```

### Cascading Replication

```
Primary ──▶ Replica-1 ──▶ Replica-2 ──▶ Replica-3

  + Reduces replication load on primary
    (only streams to Replica-1, not all)
  - Replication lag compounds at each hop
  - Replica-1 dies → Replica-2 and 3 lose sync
  Used: very large deployments, cross-region
```

### Multi-Primary (Active-Active writes) — Advanced

```
  ┌──────────┐  ←──────────▶  ┌──────────┐
  │Primary A │  bidirectional  │Primary B │
  │ Region 1 │   replication   │ Region 2 │
  └──────────┘                 └──────────┘

  Both accept writes. Conflict resolution needed.
  Used by: CockroachDB, Cassandra, DynamoDB Global Tables
  Very complex. Avoid unless cross-region writes required.
```

---

## ShopSphere Redundancy Architecture

```
                        ┌─────────────────────────────────────────┐
                        │         AWS Multi-AZ                    │
                        │                                         │
                        │  us-east-1a         us-east-1b          │
                        │  ───────────        ───────────         │
                        │  Gateway-1          Gateway-2           │  Active-Active
                        │  order-svc ×2       order-svc ×1        │  Active-Active
                        │  PG Primary ──WAL──▶PG Standby          │  Active-Passive
                        │  Redis Master──────▶Redis Replica       │  Active-Passive
                        │  Kafka Broker 1,2   Kafka Broker 3      │  Active-Active
                        │  ES Node 1,2        ES Node 3           │  Active-Active
                        └─────────────────────────────────────────┘

Component             Pattern          Failover time    Data loss risk
──────────────────────────────────────────────────────────────────────
API Gateway           Active-Active    0s (LB reroute)  None (stateless)
Microservices         Active-Active    0s (LB reroute)  None (stateless)
PostgreSQL            Active-Passive   ~60s (RDS)       Minimal (async repl)
Redis                 Active-Passive   10-30s (Sentinel) Minimal
Kafka                 Active-Active    seconds (elect)  Zero (acks=all)
Elasticsearch         Active-Active    seconds          None
```

---

## Interview Angles

**Q: Active-Active vs Active-Passive — when to use which?**
> Stateless components (app servers, API gateways) → active-active. They hold no data, any instance is identical, no conflicts possible. Stateful components (databases) → active-passive by default. Two nodes accepting writes creates conflict risk (split-brain). Use active-active for databases only with systems built for it (CockroachDB, Cassandra) that have built-in conflict resolution.

**Q: What is replication lag and why does it matter?**
> Replication lag is the delay between a write on the primary and that write appearing on the replica. During this window, a read from the replica returns stale data. This matters for read-your-writes consistency — a user who just placed an order shouldn't see it missing from their order list because their read hit a lagging replica.

**Q: What is split-brain and how do systems prevent it?**
> Split-brain occurs when a network partition causes two nodes to simultaneously believe they're the primary and both accept writes. When the partition heals, their states have diverged — two conflicting truths. Prevention uses quorum: a node can only become primary if it gets acknowledgment from a majority of nodes (or an external arbiter like etcd). The minority partition can't get quorum, so it stays passive.

**Q: What's the ISR in Kafka and why does it matter?**
> ISR (In-Sync Replicas) is the set of replicas fully caught up with the leader's log. With `acks=all`, the producer only receives acknowledgment once all ISR members have persisted the message. This guarantees zero data loss even if the leader dies immediately after — any surviving ISR member has the complete data and can be safely elected leader.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  Active-Passive  │ One primary serves traffic, standby is synced     │
│                  │ but silent. Clear authority, no conflict risk.     │
│                  │ Best for: databases, stateful stores.             │
├──────────────────────────────────────────────────────────────────────┤
│  Active-Active   │ All nodes serve traffic simultaneously.           │
│                  │ Zero failover time. Risk: write conflicts.        │
│                  │ Best for: stateless services, caches, queues.     │
├──────────────────────────────────────────────────────────────────────┤
│  Replication     │ PG → WAL streaming. Redis → command stream.       │
│  mechanisms      │ Kafka → ISR log replication.                      │
├──────────────────────────────────────────────────────────────────────┤
│  Split-brain     │ Two primaries diverge during network partition.   │
│  prevention      │ Quorum/consensus (etcd, Raft, Sentinels) ensures  │
│                  │ only majority partition can elect a primary.      │
├──────────────────────────────────────────────────────────────────────┤
│  ShopSphere      │ Services = AA. PostgreSQL/Redis = AP.             │
│                  │ Kafka/ES = AA. Multi-AZ for real independence.    │
└──────────────────────────────────────────────────────────────────────┘
```

---

Solid foundation now: scaling, statelessness, rate limiting, SPOFs, and replication all done. Next is **Topic 6 — Thundering Herd Problem** — one of the most interesting failure modes in distributed systems, where doing the right thing (caching) creates a new catastrophic failure pattern. Ready?
