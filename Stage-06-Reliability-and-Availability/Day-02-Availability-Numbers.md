# Stage 6 — Reliability & Availability
## Topic 2 : Availability Numbers

---

### Why Do Availability Numbers Matter?

Before writing a single line of architecture, senior engineers ask:

> **"What's the required availability of this system?"**

Because the answer **completely changes** your architecture, cost, and complexity.

99% availability sounds great. It's actually **3.65 days of downtime per year.** That's a fired-able offense at any serious company.

---

### The Nines Table — Memorize This

| Availability | Downtime / Year | Downtime / Month | Downtime / Week | Tier |
|---|---|---|---|---|
| **99%** | 3.65 days | 7.2 hours | 1.68 hours | Hobby |
| **99.9%** | 8.76 hours | 43.8 minutes | 10.1 minutes | Startup |
| **99.95%** | 4.38 hours | 21.9 minutes | 5 minutes | Business |
| **99.99%** | 52.6 minutes | 4.38 minutes | 1.01 minutes | Enterprise |
| **99.999%** | 5.26 minutes | 26.3 seconds | 6.05 seconds | Carrier-grade |
| **99.9999%** | 31.5 seconds | 2.6 seconds | 0.6 seconds | God-tier |

---

### How to Calculate It

```
Availability % = (Total Time - Downtime) / Total Time × 100

One year = 365 × 24 × 60 = 525,600 minutes

99.9%   → 0.001 × 525,600 = 525.6 minutes  = ~8.76 hours
99.99%  → 0.0001 × 525,600 = 52.56 minutes
99.999% → 0.00001 × 525,600 = 5.256 minutes
```

---

### What Does Each Tier Actually Mean?

#### 99.9% — "Three Nines" — Startup / Internal Tools
```
   Jan ████████████████████████████████ ~44 min downtime this month
   Feb ████████████████████████████████ ~44 min
   ...
   Total: 8.76 hours/year

   Reality: Restart during deploy = fine
            Weekend maintenance window = fine
            Single DB instance = okay-ish
```

#### 99.99% — "Four Nines" — Enterprise / SaaS Products
```
   Entire month budget: 4.38 minutes

   Deploy takes 3 minutes? → You just burned 68% of monthly budget.

   Reality: Zero-downtime deploys are MANDATORY
            Multi-AZ database required
            Auto-healing infrastructure required
```

#### 99.999% — "Five Nines" — Telecom / Banking / Payments
```
   Entire year budget: 5.26 minutes

   Reality: Chaos engineering is a job role here
            Every component is redundant × 3
            Even the redundancy has redundancy
            Google, AWS, Visa operate here
```

---

### The Hidden Math — Serial vs Parallel Availability

This is the **most asked interview concept** around availability numbers.

#### Serial (chained) components — availability MULTIPLIES DOWN

```
Request must pass through ALL components:

  [Load Balancer] → [App Server] → [Database] → [Cache]
      99.99%            99.9%         99.9%       99.9%

  Total = 0.9999 × 0.999 × 0.999 × 0.999
        = 0.9999 × 0.997
        = 0.9969
        = 99.69%  ← worse than any single component!
```

> **Every component you add in series REDUCES overall availability.**

#### Parallel (redundant) components — availability IMPROVES

```
Both replicas must fail simultaneously for the system to fail:

  Probability(both fail) = (1 - 0.999) × (1 - 0.999)
                         = 0.001 × 0.001
                         = 0.000001

  Availability = 1 - 0.000001 = 99.9999%  🎉
```

```
              ┌─── DB Replica 1 (99.9%) ───┐
  Request ───►│                             ├──► Response
              └─── DB Replica 2 (99.9%) ───┘

  Both must be down simultaneously for failure.
  That's (0.001)² = 0.000001 probability.
```

> **Redundancy is how you climb the nines. Parallelism multiplies reliability.**

---

### Real Architecture Decisions Per Tier

```
99.9%  ──► Single region, single AZ
           Rolling deploys okay
           Manual failover acceptable
           RTO: hours, RPO: minutes

99.99% ──► Multi-AZ within one region
           Blue-green / canary deploys mandatory
           Automated failover required
           Health checks on every node
           RTO: minutes, RPO: seconds

99.999%──► Multi-region active-active
           Chaos engineering in prod
           Auto-scaling at every layer
           Global load balancing (Route53, Cloudflare)
           RTO: seconds, RPO: near-zero
```

---

### RTO vs RPO — Critical Definitions

These always come paired with availability discussions.

```
Failure occurs
     │
     ▼
─────●──────────────────────────────────────────────►  time
     │◄────────── RTO ──────────────────►│
                                         ▼
                                   System restored

     │◄── RPO ──►│
   Last          Failure
  backup         point

RPO = Recovery Point Objective  → How much DATA can you lose?
RTO = Recovery Time Objective   → How long can you be DOWN?
```

| Tier | RTO | RPO |
|---|---|---|
| Startup | Hours | Hours |
| Business | 30 min | 15 min |
| Enterprise | < 1 min | Near zero |
| Five Nines | Seconds | Zero |

---

### Planned vs Unplanned Downtime

Most teams forget that **deploys count as downtime** unless you design around them.

```
Downtime sources:
  ├── Planned
  │     ├── Deployments         ← zero-downtime deploy solves this
  │     ├── DB migrations       ← backward-compatible migrations
  │     └── Maintenance windows ← avoid if 99.99%+
  │
  └── Unplanned
        ├── Hardware failure    ← solved by redundancy
        ├── Software bugs       ← solved by canary + rollback
        ├── Cascade failures    ← solved by circuit breakers (Topic 5)
        └── Network partition   ← solved by multi-AZ / region
```

---

### ShopSphere Availability Targets 🛒

| Service | Target | Reason |
|---|---|---|
| **Payment Service** | 99.999% | Money movement, zero tolerance |
| **Order Service** | 99.99% | Revenue critical |
| **Product Catalog** | 99.9% | Stale data for seconds = fine |
| **Search Service** | 99.9% | Fallback to DB scan possible |
| **Notification Service** | 99.9% | Email delay acceptable |
| **Review Service** | 99.5% | Non-critical path |
| **Admin Dashboard** | 99% | Internal tool |

**Architecture implication for Payment + Order:**
```
  ┌──────────────┐    ┌──────────────┐
  │  App Server  │    │  App Server  │    ← Multi-instance
  │  (AZ-1)      │    │  (AZ-2)      │
  └──────┬───────┘    └──────┬───────┘
         │                   │
  ┌──────▼───────────────────▼───────┐
  │         Primary DB (AZ-1)        │
  │  ──────────────────────────────  │
  │         Replica  DB (AZ-2)       │    ← Synchronous replication
  └──────────────────────────────────┘
         Automated failover < 30s
```

---

### Interview Questions 🎯

**Q1. Your service has 99.9% availability. You add 3 more dependencies each at 99.9%. What's total availability?**
> 0.999⁴ = 0.996 = **99.6%**. Each added dependency in series degrades overall availability. This is why microservices need circuit breakers and timeouts — to prevent cascading failures.

**Q2. How do you achieve 99.99% without changing your DB code?**
> Add a read replica in a second AZ with automated failover. Use a load balancer with health checks. Switch to rolling/blue-green deploys. Each removes a downtime source.

**Q3. Difference between RTO and RPO?**
> RTO = how long the system can be down before business impact becomes unacceptable. RPO = how much data loss is acceptable. A backup every hour = RPO of 1 hour. Synchronous replication = RPO near zero.

**Q4. A system has 99.999% availability. How many minutes of downtime per year is the budget?**
> 5.26 minutes. That's it. A single bad deploy that takes 10 minutes to rollback **blows the annual budget twice over**.

**Q5. How does AWS achieve 99.99%+ SLAs?**
> Multi-AZ deployments, automatic failover, health-check-driven routing, chaos engineering, independent failure domains per AZ, and global redundancy for their core infrastructure services.

---

### One-Line Summary

> **Each "nine" is 10× harder to achieve than the last. In series, availability multiplies down — add redundancy in parallel to climb back up. Know your RTO/RPO before picking your architecture.**

---

Ready for **Topic 3: Replication** — synchronous vs asynchronous, how replicas work under the hood, and what "replication lag" actually costs you?

Type **next** 🚀
