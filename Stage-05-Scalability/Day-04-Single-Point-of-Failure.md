# Topic 4 — Single Point of Failure (SPOF)

## What Is a SPOF?

```
A Single Point of Failure is any component in your system
whose failure causes the ENTIRE system (or a critical part) to fail.

The test:
  "If THIS component dies right now —
   what percentage of users are affected?"

  100% affected → SPOF
  Partial degradation → not a SPOF (but still a risk)
```

---

## Why SPOFs Are Inevitable at First

Every system starts with SPOFs. A startup with 100 users doesn't need redundancy. The engineering challenge is **identifying SPOFs before they kill you in production** and eliminating them proportional to the cost of failure.

```
Cost of failure = Probability of failure × Impact of failure

  Probability: How often does this component fail?
  Impact:      How many users / how much revenue is lost per minute?

High probability + High impact = fix NOW
Low probability  + High impact = fix before scale
High probability + Low impact  = monitor, fix eventually
Low probability  + Low impact  = acceptable risk
```

---

## Identifying SPOFs — The Dependency Walk

Draw your system, then ask: **what happens if each box dies?**

```
ShopSphere — naive first architecture:

  Users
    │
    ▼
  ┌──────────────┐
  │  API Gateway │  ← SPOF #1: dies → entire platform down
  └──────────────┘
         │
    ┌────┴──────────────────────┐
    ▼                           ▼
  ┌───────────────┐    ┌──────────────────┐
  │ order-service │    │ product-service  │
  └───────────────┘    └──────────────────┘
         │                      │
         ▼                      ▼
  ┌──────────────┐      ┌──────────────┐
  │  PostgreSQL  │      │    Redis     │
  │  (1 node)   │      │   (1 node)   │
  └──────────────┘      └──────────────┘
       SPOF #2               SPOF #3
  DB dies → no orders   Cache dies → product-svc crashes
                         if it doesn't handle Redis failure

         │
         ▼
  ┌──────────────┐
  │    Kafka     │  ← SPOF #4: 1 broker → events lost
  │  (1 broker)  │
  └──────────────┘
```

Every single-node component is a SPOF. Let's go through each category.

---

## SPOF Category 1 — Load Balancer / Gateway

```
Problem:
  ┌──────────┐
  │  Client  │──▶ [ API Gateway — 1 instance ] ──▶ services
  └──────────┘           ↑
                     DIES → 100% users down

Fix — Multiple Gateway Instances + External LB:

  ┌──────────┐
  │  Client  │──▶ [ AWS ALB / Nginx / Cloudflare ]
  └──────────┘              │
                   ┌────────┴────────┐
                   ▼                 ▼
             [ Gateway-1 ]    [ Gateway-2 ]
                   │                 │
                   └────────┬────────┘
                             ▼
                         Services
```

The **external load balancer** (AWS ALB, Cloudflare) is itself managed infrastructure with built-in redundancy — you don't manage it. It distributes across your gateway instances. If Gateway-1 dies, ALB stops routing to it automatically via health checks.

```
Health Check Flow:
  ALB pings GET /actuator/health every 10s
  Gateway-1 stops responding → ALB marks it unhealthy
  All traffic → Gateway-2 within ~30 seconds
  Gateway-1 recovers → ALB adds it back
```

---

## SPOF Category 2 — Database (Hardest One)

The database is the hardest SPOF to eliminate because it's **stateful**. You can't just run two copies independently — they'd diverge.

```
Problem:
  ┌──────────────────┐
  │  PostgreSQL      │
  │  Single Node     │  ← Dies → order-service can't read or write
  │  host: pg-01     │    All orders, users, products inaccessible
  └──────────────────┘

Two separate failure modes:
  1. Read-heavy load: too many SELECT queries
  2. Write failure / node crash: DB completely down
```

**Fix — Primary + Read Replicas:**

```
Writes (INSERT, UPDATE, DELETE):
  order-service ──▶ [ PostgreSQL Primary ]
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
             [ Replica-1 ]       [ Replica-2 ]
                    ↑                    ↑
Reads (SELECT):     └──────────┬─────────┘
  order-service ───────────────┘
  (read from replicas, takes load off primary)

Replication:
  Primary streams WAL (Write-Ahead Log) to replicas
  Replicas apply the same changes → eventual consistency
  Replication lag: typically < 100ms on same region
```

```
Failover (Primary dies):
  BEFORE (manual era):
    DBA wakes up at 3 AM, manually promotes replica → 20 min downtime

  NOW (automated):
    AWS RDS Multi-AZ: standby promoted in < 60 seconds automatically
    Patroni (self-hosted): etcd-based leader election, auto-promotion
    PgBouncer: connection pooler transparently reroutes connections
```

**The replication lag problem:**
```
t=0ms:  User places order → written to Primary ✅
t=50ms: Replication lag — replica not yet updated
t=30ms: User refreshes order list → reads from replica
        → order not visible yet 😱

Fix options:
  1. Read-your-writes consistency: after a write,
     read from primary for next N seconds for same user
  2. Session stickiness to primary for critical reads
  3. Accept eventual consistency for non-critical reads
     (product catalog, reviews — stale for 100ms is fine)
```

---

## SPOF Category 3 — Cache (Redis)

```
Problem:
  Redis dies → product-service has no cache
  
  Two failure modes depending on how you coded it:

  Mode A (cache as required dependency):
    productService.getProduct(id) {
        return redis.get(id);  // throws exception if Redis down
    }
    → Service crashes entirely ❌

  Mode B (cache as optional layer — correct):
    productService.getProduct(id) {
        try {
            val cached = redis.get(id);
            if (cached != null) return cached;
        } catch (RedisException e) {
            log.warn("Redis down, falling back to DB");
        }
        return database.findById(id);  // fallback ✅
    }
    → Performance degrades (DB hit every time) but service stays up
```

**Fix — Redis Sentinel or Redis Cluster:**

```
Redis Sentinel (High Availability):

  ┌──────────────┐
  │ Redis Master │ ← writes go here
  └──────────────┘
         │ replication
    ┌────┴────────┐
    ▼             ▼
[ Replica-1 ] [ Replica-2 ]

  + 3 Sentinel processes monitoring all nodes
  Master dies → Sentinels vote → promote a replica → update DNS
  App reconnects to new master automatically via Sentinel client

Redis Cluster (Sharding + HA):
  16384 hash slots split across N masters
  Each master has replicas
  Any master dies → its replica promoted
  Used when data > single node capacity
```

---

## SPOF Category 4 — Message Broker (Kafka)

```
Problem:
  Single Kafka broker:
    Producer → [ Broker-1 ] → Consumer
    Broker-1 dies → no events → notification-svc gets nothing
    Events in-flight: LOST (if not persisted) or STUCK

Fix — Kafka Cluster with Replication:

  3 brokers, replication factor = 3

  Topic: order-events, Partition 0:
    Leader:   Broker-1  ← producer writes here
    Replica:  Broker-2  ← synced copy
    Replica:  Broker-3  ← synced copy

  Broker-1 dies:
    Controller (ZooKeeper/KRaft) detects failure
    Broker-2 elected as new leader for partition 0
    Producers and consumers redirect to Broker-2
    Zero message loss (ISR — In-Sync Replicas)
```

```
Key Kafka durability settings:
  replication.factor = 3       ← 3 copies of each partition
  min.insync.replicas = 2      ← at least 2 must confirm write
  acks = all                   ← producer waits for all replicas

  With these: can lose 1 broker, zero data loss ✅
```

---

## SPOF Category 5 — The Entire Availability Zone

This is the one most engineers forget until it's too late.

```
All your redundant components in ONE data center:

  AWS us-east-1a:
    Gateway-1, Gateway-2
    order-svc × 3
    PostgreSQL Primary + Replica
    Redis Sentinel cluster
    Kafka cluster (3 brokers)

  us-east-1a has a power outage (happened to AWS in 2011, 2017, 2023)
  → EVERYTHING dies simultaneously → all redundancy was worthless

Fix — Multi-AZ Deployment:

  us-east-1a                us-east-1b
  ─────────────────         ─────────────────
  Gateway-1                 Gateway-2
  order-svc-1, 2            order-svc-3, 4
  PostgreSQL Primary        PostgreSQL Replica (standby)
  Redis Master              Redis Replica
  Kafka Broker-1, 2         Kafka Broker-3

  AZ goes down → other AZ absorbs all traffic
  AWS ALB routes only to healthy AZ automatically
```

---

## SPOF Category 6 — Humans and Processes

Often overlooked. Non-technical SPOFs:

```
Human SPOFs:
  "Only Ravi knows the DB password" → Ravi on vacation → no DB access
  "Only the founder can approve production deploys" → bottleneck

Process SPOFs:
  Manual deployments → someone has to do them → error-prone
  No runbooks → incident response depends on tribal knowledge

Fix:
  Secrets in Vault / AWS Secrets Manager (not in one person's head)
  Automated CI/CD (no manual steps for deploys)
  Documented runbooks for every known failure scenario
  On-call rotation (not one person)
```

---

## ShopSphere — SPOF Audit and Fixes

```
Component          Current State      SPOF?   Fix
────────────────────────────────────────────────────────────────────
API Gateway        1 instance         YES     Multiple instances + AWS ALB
PostgreSQL         1 node             YES     Primary + replica, Multi-AZ RDS
Redis              1 node             YES     Redis Sentinel, fallback to DB
Kafka              1 broker           YES     3-broker cluster, RF=3
Each microservice  1 pod in K8s       YES     replicas: 3 in Deployment spec
Elasticsearch      1 node             YES     3-node cluster, 1 replica shard
RabbitMQ           1 node             YES     Mirrored queues or Quorum queues

After fixes:
  No single component failure → total outage
  Any one node dies → degraded performance at worst
  Auto-recovery via K8s, RDS Multi-AZ, Kafka controller
```

**K8s Deployment for order-service with SPOF eliminated:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3                    # 3 pods, no SPOF
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1          # at most 1 pod down during deploy
      maxSurge: 1
  template:
    spec:
      affinity:
        podAntiAffinity:         # ← CRITICAL
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: order-service
              topologyKey: kubernetes.io/hostname
      # Forces pods onto DIFFERENT nodes
      # If all 3 pods land on same node → node dies → SPOF again
```

Without `podAntiAffinity`, K8s might schedule all 3 replicas on the same node — you have 3 pods but still one point of failure.

---

## The SPOF Elimination Checklist

```
For every component, ask:

  □ What happens if this dies?
  □ Is there an automatic failover, or manual intervention needed?
  □ How long does failover take? (RTO — Recovery Time Objective)
  □ Is any data lost during failover? (RPO — Recovery Point Objective)
  □ Are the redundant components in different AZs / nodes?
  □ Does the application handle the failure gracefully (fallback)?
  □ Is there a health check that detects failure automatically?
  □ Is the runbook documented for when this fails?
```

---

## Interview Angles

**Q: How do you identify SPOFs in a system design?**
> Walk the request path end to end. For every component — load balancer, app service, database, cache, message broker — ask: if this dies right now, what percentage of users are affected? 100% impact = SPOF. Then check if redundant components are truly independent (different nodes, different AZs) or just logically separate but physically co-located.

**Q: My database is still a SPOF even with replicas — why?**
> The primary is still a single write node. If it dies and failover takes 60 seconds, that's 60 seconds of write downtime. With async replication, those last few seconds of writes may be lost (RPO > 0). True zero-SPOF for databases requires synchronous replication (performance cost) or distributed databases like CockroachDB / Vitess that handle split-brain automatically.

**Q: What's the difference between SPOF elimination and redundancy?**
> They're related but different. Eliminating a SPOF means the failure of one component no longer causes full system failure. Redundancy is the mechanism — running multiple copies. But redundancy alone isn't enough: if all replicas are on the same node or same AZ, you've added redundancy without eliminating the SPOF (the node/AZ itself is still the SPOF).

**Q: What's podAntiAffinity in Kubernetes and why does it matter for SPOFs?**
> Without it, K8s might schedule all replicas of a service on the same node for bin-packing efficiency. If that node dies, all pods go down simultaneously — despite having 3 replicas. PodAntiAffinity forces replicas onto different nodes (or different AZs), ensuring node failure only takes down one replica, not all.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│  SPOF Test     │  "If this dies → what % of users are affected?"    │
├─────────────────────────────────────────────────────────────────────┤
│  Common SPOFs  │  Gateway, DB primary, Cache, Broker, Single AZ     │
├─────────────────────────────────────────────────────────────────────┤
│  Fixes         │  Multiple instances + LB, DB replicas + failover,  │
│                │  Redis Sentinel, Kafka RF=3, Multi-AZ, Anti-affinity│
├─────────────────────────────────────────────────────────────────────┤
│  Not enough    │  Redundancy on same node/AZ = still a SPOF         │
├─────────────────────────────────────────────────────────────────────┤
│  ShopSphere    │  Every single-node component is a SPOF today.      │
│                │  Fix: K8s replicas + anti-affinity, RDS Multi-AZ,  │
│                │  Redis Sentinel, Kafka 3-broker cluster             │
└─────────────────────────────────────────────────────────────────────┘
```

---

SPOF done — and this feeds directly into the next topic. **Topic 5 — Redundancy and Replication** goes deeper: active-active vs active-passive, what exactly gets replicated, how replication works under the hood, and the tradeoffs. Ready?
