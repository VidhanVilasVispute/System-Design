# Stage 6 — Reliability & Availability
## Topic 4 : Failover

---

### What is Failover?

> **Failover = the process of automatically promoting a replica to primary when the primary becomes unavailable — with zero (or minimal) human intervention.**

Without failover:
```
Primary dies at 3:00 AM
On-call engineer wakes up at 3:47 AM
Manually promotes replica at 4:12 AM
System was down for 72 minutes ← unacceptable
```

With failover:
```
Primary dies at 3:00 AM
Failover system detects at 3:00:08 AM
Replica promoted at 3:00:35 AM
System restored in 35 seconds ← acceptable
```

---

### The Failover Pipeline — Step by Step

```
Step 1: DETECTION
  ┌─────────────┐
  │  Primary    │  ← stops responding
  │  (dead)     │
  └─────────────┘
        ▲
        │  heartbeat timeout (e.g. 10s)
        │
  ┌─────────────┐
  │  Monitor    │  ← Sentinel / Patroni / Orchestrator
  │  Agent      │
  └─────────────┘

Step 2: CONFIRMATION (avoid false positives)
  Monitor pings Primary from multiple agents
  2 of 3 agents can't reach Primary → confirmed dead
  (not just a network blip between monitor and primary)

Step 3: ELECTION
  Which replica becomes new Primary?
  → Most up-to-date WAL position wins
  → Highest priority configured wins
  → First to acquire leadership lock wins

Step 4: PROMOTION
  Replica stops accepting replication stream
  Replica opens up for writes
  Replica announces itself as new Primary

Step 5: REDIRECT
  DNS updated  OR  Load balancer config updated
  App connections re-pointed to new Primary
  Old Primary connections drained / rejected
```

---

### Failover Detection — The Timing Problem

This is where most systems get it wrong.

```
Timeline of a Primary failure:

t=0s    Primary process crashes
t=0s    Existing connections → get TCP RST immediately ✅ (fast)
t=0s    New connections → get "Connection Refused" ✅ (fast)

BUT:

t=0s    Primary host itself hangs (not crash, but freeze)
t=0s    TCP connections stay open but get no response
t=30s   TCP keepalive timeout fires
t=30s   Client finally sees error ← 30 seconds of hanging!
```

**Heartbeat-based detection:**
```
Monitor → Primary: "ping" every 2 seconds
Primary → Monitor: "pong"

If 5 consecutive pings fail → Primary declared dead
Detection time = 5 × 2s = 10 seconds

But: Is 10 seconds a crash or a network hiccup?
     Too aggressive → false positives → unnecessary failovers
     Too conservative → longer downtime
```

**Production tuning (PostgreSQL + Patroni):**
```yaml
# patroni.yml
ttl: 30                  # leader lock expiry (seconds)
loop_wait: 10            # how often Patroni checks
retry_timeout: 10        # how long to retry before giving up
maximum_lag_on_failover: 1048576  # 1MB — don't promote a
                                   # replica too far behind
```

---

### The Split-Brain Problem — Most Dangerous Failure Mode

> **Split-brain = two nodes both believe they are Primary simultaneously.**

```
Normal:
  Primary ──────────────── Replica
  (writes)    network       (reads)

Network partition occurs:
  Primary ──── ✂ ──────── Replica

  Primary thinks: "Replica is dead, I'm still Primary"
  Replica thinks: "Primary is dead, I'll promote myself"

  Now BOTH accept writes:

  Primary: order #101 → user A gets product X
  Replica: order #101 → user B gets product X  ← SAME ORDER ID!

  Network heals → CONFLICT CHAOS
  Two different realities must be merged 💀
```

**How systems prevent split-brain:**

#### 1. STONITH — Shoot The Other Node In The Head
```
Before Replica promotes itself:
  → Send "fence" command to Primary's power controller
  → Forcibly shut down Primary's VM / machine
  → Now only ONE node is alive → safe to promote

Used in: Pacemaker/Corosync, enterprise HA clusters
```

#### 2. Quorum / Majority Voting
```
5-node cluster:
  Node1(Primary), Node2, Node3, Node4, Node5

Partition: [Node1] vs [Node2, Node3, Node4, Node5]

  Node1: "I can only reach myself (1 of 5) → step down"
  Node2-5: "We have majority (4 of 5) → elect new Primary"

Rule: Only partition with MAJORITY (>50%) can elect Primary
      Minority partition goes read-only or shuts down

Used in: etcd, ZooKeeper, Raft consensus
```

#### 3. Lease / Lock-based
```
Primary must hold a "leader lease" in distributed lock store
  (etcd / ZooKeeper / Consul)

Lease expires every 10 seconds — Primary must renew it.

  Primary crashes → lease expires → replica acquires lease
  → promotes safely

  Even if old Primary comes back:
    → tries to renew lease → fails (replica holds it)
    → old Primary steps down

Used in: Patroni (PostgreSQL), Redis Sentinel
```

---

### Failover Types

#### Cold Standby
```
Primary  [ACTIVE]
Standby  [OFF / sleeping]

Failover: Boot up standby → restore latest backup → go live
Time: Minutes to hours
RPO: Hours (last backup)
RTO: Hours
Use: Non-critical systems, low cost
```

#### Warm Standby
```
Primary  [ACTIVE, taking writes]
Standby  [RUNNING, receiving async replication, NOT serving traffic]

Failover: Promote standby → redirect traffic
Time: 30 seconds to minutes
RPO: Seconds (async lag)
RTO: Minutes
Use: Most production databases
```

#### Hot Standby (Active-Active)
```
Primary  [ACTIVE, writes + reads]
Replica  [ACTIVE, reads only, in sync]

Failover: Replica already running → promote instantly
Time: Seconds
RPO: Near-zero (sync) or seconds (async)
RTO: Seconds
Use: High availability production systems
```

```
Comparison:
  Cold    ──────────────────────────── Cheapest, slowest
  Warm    ──────────────── Middle ground
  Hot     ──── Most expensive, fastest  (ShopSphere target)
```

---

### Connection Handling During Failover

Your app is connected to the (now dead) Primary. What happens?

```
Without proper handling:
  App → Primary (dead) → connection hangs 30s → exception
  App → retries → hits old Primary IP → fails again
  App → finally gets DNS update → reconnects ← 60-90 seconds

With proper handling:
  App → Primary (dead) → connection error detected quickly
  App → reads new Primary from service discovery / DNS
  App → reconnects → continues ← 5-10 seconds
```

**Spring Boot / HikariCP config for fast failover:**
```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 3000          # 3s to get connection
      keepalive-time: 30000             # send keepalive every 30s
      max-lifetime: 1800000             # recycle connections every 30min
      connection-test-query: SELECT 1   # validate before use
      
  # For multi-datasource (primary + replica routing)
  # Use Spring's @Primary / @Transactional(readOnly) routing
  # On failover → replica becomes primary → both point to same
```

**DNS TTL — the silent failover killer:**
```
Your DB endpoint: db.shopsphere.internal → 10.0.1.5 (Primary)

Primary dies → failover → new Primary at 10.0.1.8
DNS updated: db.shopsphere.internal → 10.0.1.8

But your app has DNS cached for TTL = 300 seconds (5 min)!
App keeps hitting 10.0.1.5 for 5 minutes → all writes fail

Fix: Set DNS TTL to 5–10 seconds for DB endpoints
     OR use a connection proxy (PgBouncer, ProxySQL)
     that handles the redirect transparently
```

---

### PgBouncer / ProxySQL — Proxy-based Failover

```
         App Servers
            │  │  │
            ▼  ▼  ▼
       ┌─────────────┐
       │  PgBouncer  │  ← Connection pooler + proxy
       │  (proxy)    │
       └──────┬──────┘
              │
     ┌────────┴────────┐
     ▼                 ▼
  Primary           Replica
  (10.0.1.5)        (10.0.1.8)

On failover:
  PgBouncer reconfigures internally
  Apps never know the IP changed
  Connection pool is re-established transparently
```

---

### ShopSphere Failover Architecture 🛒

```
┌───────────────────────────────────────────────────────┐
│                    ShopSphere HA                       │
│                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐ │
│  │ Order    │    │ Payment  │    │   Patroni        │ │
│  │ Service  │    │ Service  │    │   (HA Manager)   │ │
│  └────┬─────┘    └────┬─────┘    └────────┬─────────┘ │
│       │               │                   │           │
│       └───────────────┴──────────────────►│           │
│                                           │ monitors  │
│                       ┌───────────────────▼─────────┐ │
│                       │         etcd cluster        │ │
│                       │  (leader lock / quorum)     │ │
│                       └───────────────┬─────────────┘ │
│                                       │               │
│              ┌────────────────────────┴─────────────┐ │
│              │                                       │ │
│     ┌────────▼──────┐  sync repl    ┌───────────┐   │ │
│     │  Primary DB   │──────────────►│ Replica   │   │ │
│     │  (AZ-1)       │               │ (AZ-2)    │   │ │
│     │  holds lease  │               │ standby   │   │ │
│     └───────────────┘               └───────────┘   │ │
│                                                      │ │
│  On failure:                                         │ │
│  1. Patroni detects Primary miss (10s)               │ │
│  2. etcd lease expires                               │ │
│  3. Replica acquires lease                           │ │
│  4. Replica promoted → new Primary                   │ │
│  5. Services reconnect via PgBouncer                 │ │
│  Total: ~30 seconds                                  │ │
└───────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What is split-brain and how do you prevent it?**
> Split-brain is when two nodes simultaneously believe they are Primary, causing conflicting writes. Prevention: use quorum-based elections (only majority partition can elect), STONITH (forcibly fence the old Primary before promoting), or lease-based locking (old Primary loses its lock before new one is granted).

**Q2. Why does DNS TTL matter during failover?**
> If DNS TTL is too high (e.g., 5 minutes), apps continue routing to the old Primary IP even after failover, causing failures until the cache expires. Fix: set DB endpoint TTL to 5–10 seconds, or use a connection proxy like PgBouncer that handles redirection transparently without DNS dependency.

**Q3. How does Patroni handle PostgreSQL failover?**
> Patroni uses etcd as a distributed lock store. The Primary holds a renewable leader lease. If the Primary fails to renew (crashes or hangs), the lease expires. Patroni agents on replicas race to acquire the lock. Winner promotes itself to Primary. Guarantees no split-brain because two nodes can't hold the same etcd key simultaneously.

**Q4. What is failover detection lag and why does it matter?**
> It's the time between Primary actually dying and the system detecting it. Too short — false positives, unnecessary failovers, data inconsistency. Too long — extended downtime. Typical production sweet spot is 10–30 seconds using heartbeat timeouts with multiple confirmation checks.

**Q5. After failover, old Primary comes back online. What happens?**
> In a correctly configured system (Patroni + etcd), the old Primary finds its lease gone, discovers a new Primary exists, and automatically rejoins as a replica — starts streaming WAL from the new Primary. It does NOT reclaim Primary status. This is called "former Primary demotion."

---

### One-Line Summary

> **Failover is automatic Primary replacement — detection lag + split-brain prevention + DNS/proxy redirection are the three hard problems. Solve them with heartbeat quorums, lease-based locking, and a connection proxy like PgBouncer.**

---

Ready for **Topic 5: Circuit Breakers** — stop calling a failing service, fail fast, state machine internals (Closed → Open → Half-Open), and how Resilience4j implements it in ShopSphere?

Type **next** 🚀
