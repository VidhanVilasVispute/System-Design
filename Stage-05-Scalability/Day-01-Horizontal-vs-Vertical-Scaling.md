Love it — Stage 5 is where system design gets *real*. Scalability is the difference between a system that works for 100 users and one that works for 10 million.

We go one topic at a time, deep. Starting with the foundation:

---

# Topic 1 — Horizontal vs Vertical Scaling

## The Core Idea

Imagine your ShopSphere `order-service` is struggling under load. You have two choices:

```
VERTICAL SCALING (Scale Up)          HORIZONTAL SCALING (Scale Out)
─────────────────────────────        ──────────────────────────────────────
        ┌──────────┐                     ┌────────┐ ┌────────┐ ┌────────┐
        │          │                     │ order  │ │ order  │ │ order  │
        │  order   │  ← 32 CPU           │  svc   │ │  svc   │ │  svc   │
        │  svc     │     128 GB RAM       │ inst-1 │ │ inst-2 │ │ inst-3 │
        │          │     NVMe SSD         └────────┘ └────────┘ └────────┘
        └──────────┘                          ↑           ↑           ↑
    One beefy machine                    Load Balancer distributes traffic
```

---

## Vertical Scaling — Theory Deep Dive

You **upgrade the existing machine**: more CPU cores, more RAM, faster disk, faster NIC.

### Why it works initially
- No code changes needed — your app suddenly has more resources
- No network hops between nodes — everything in one process/machine
- Simple ops: one machine to monitor, patch, restart

### The hard limits
```
CPU Scaling:
  2 cores → 4 cores → 8 → 16 → 32 → 64 → 128...
  Cost doubles every step, but performance gain shrinks (Amdahl's Law)

RAM:
  Physical slot limits. A typical server board maxes at 1–4 TB.

Cost Curve (rough reality):
  ┌─────────────────────────────────────────────────┐
  │  Cost                                           │
  │    ↑                              ╭─────        │
  │    │                         ╭───╯             │
  │    │                   ╭────╯                  │
  │    │          ╭────────╯                       │
  │    │  ╭───────╯                                │
  │    └──┴────────────────────────────→ Capacity  │
  │   Exponential cost for linear gain             │
  └─────────────────────────────────────────────────┘
```

### Amdahl's Law (why vertical hits a wall)
If 20% of your code is single-threaded (non-parallelizable):
```
Max Speedup = 1 / (0.20 + (0.80 / N cores))

N=1:   1x
N=4:   2.5x
N=16:  3.6x
N=64:  4.5x
N=∞:   5x  ← you can NEVER exceed this, no matter how many cores
```

**The point:** Even infinite hardware can't fix serial bottlenecks.

---

## Horizontal Scaling — Theory Deep Dive

You **add more machines** running the same service, and a load balancer distributes traffic.

### The architecture
```
                         ┌─────────────────────┐
  Client Requests ──────▶│    Load Balancer     │
                         │  (e.g. Nginx/AWS ALB)│
                         └─────────────────────┘
                           /        |         \
                          /         |          \
                   ┌──────┐   ┌──────┐   ┌──────┐
                   │order │   │order │   │order │
                   │svc-1 │   │svc-2 │   │svc-3 │
                   │:8080 │   │:8080 │   │:8080 │
                   └──────┘   └──────┘   └──────┘
                         \        |        /
                          \       |       /
                       ┌───────────────────┐
                       │    Shared DB /    │
                       │  Redis / Kafka    │
                       └───────────────────┘
```

### Why horizontal wins long-term — 5 reasons

**1. Linear cost scaling**
- 10 small instances at ₹500/mo = ₹5000/mo
- 1 giant instance with same throughput = ₹15,000/mo (premium pricing for top-tier hardware)

**2. No single upper bound**
- Vertical: hits hardware ceiling
- Horizontal: add another node → scale infinitely (in theory)

**3. Fault tolerance is built-in**
```
Vertical:
  Machine dies → 100% downtime

Horizontal:
  1 of 3 nodes dies → 66% capacity, still serving traffic
  LB detects failure via health checks, routes away
```

**4. Zero-downtime deployments become possible**
```
Rolling Deploy (horizontal only):
  Step 1: Take svc-1 out of rotation, deploy new version
  Step 2: Health check passes → svc-1 back in rotation
  Step 3: Repeat for svc-2, svc-3
  Result: Users never notice the deployment
```

**5. Elastic scaling — pay for what you use**
```
Black Friday 11 PM:
  10 order-service instances spinning

3 AM Tuesday:
  2 order-service instances → 80% cost savings
```

---

## The Critical Prerequisite: Statelessness

Horizontal scaling only works if instances are **interchangeable**. If svc-1 holds a user's session in memory and the next request hits svc-2 — the user gets logged out. This is the **stateful trap**.

```
BROKEN (stateful):
  Request 1 → svc-1 (session stored in memory of svc-1)
  Request 2 → svc-2 (no session here) → user sees error

CORRECT (stateless):
  Request 1 → svc-1 (reads session from Redis)
  Request 2 → svc-2 (reads same session from Redis) → works fine
```

We'll go deep on this in **Topic 2 (Stateless Services)** — but know that it's the *prerequisite* for everything in this stage.

---

## ShopSphere Mapping

Every single one of your services should be horizontally scalable. Let's check which ones naturally are and which need care:

```
Service              State Risk?            How ShopSphere handles it
──────────────────────────────────────────────────────────────────────
user-service         JWT = stateless ✅     No server-side session
order-service        Order state in DB ✅   PostgreSQL is source of truth
product-service      Catalog from DB ✅     Redis cache (shared, not local)
search-service       ES cluster ✅          Elasticsearch handles its own sharding
notification-svc     Async via RabbitMQ ✅  Queue decouples producers/consumers
review-service       DB-backed ✅           Fine
API Gateway          Routing only ✅        Pure stateless proxy

Risk area: If any service cached something IN-MEMORY (like a local HashMap
           of logged-in users), that breaks horizontal scaling.
           ShopSphere avoids this by using Redis for all shared state.
```

---

## When to use Vertical Scaling

Horizontal isn't always the answer. Vertical scaling makes sense when:

| Scenario | Why Vertical |
|---|---|
| Databases (e.g. PostgreSQL primary) | Distributed writes are extremely complex. Scale vertically first, then read replicas |
| ML model inference (single GPU) | Can't easily shard across GPUs for one inference call |
| Legacy monolith you can't refactor | No option to run multiple instances safely |
| Early stage startup (< 1000 users) | Operational simplicity beats theoretical scale |

> **The rule of thumb:** Scale vertically until it hurts, then refactor for horizontal.

---

## Interview Angles

**Q: Why can't you just keep vertically scaling?**
> Hardware has physical ceilings. Cost grows exponentially while performance grows sub-linearly (Amdahl's Law). You also get a SPOF — one machine dies, everything dies.

**Q: What's the main challenge with horizontal scaling?**
> Statefulness. All shared state must live outside the application instances — in a database, cache, or message queue. The instances must be truly interchangeable.

**Q: How does ShopSphere's order-service scale horizontally?**
> It's stateless — no in-memory session. All state is in PostgreSQL. Redis handles distributed caching. Kafka handles async messaging. Multiple instances can run behind a load balancer (or K8s HPA) without any coordination between them.

**Q: Database is still a single machine — isn't that a bottleneck?**
> Yes, this is the classic "stateless app, stateful DB" problem. You solve it with read replicas for read scaling, sharding for write scaling, and caching (Redis) to reduce DB pressure. We'll cover this in later stages.

---

## Summary

```
┌────────────────────┬──────────────────────────┬──────────────────────────┐
│                    │   Vertical Scaling        │   Horizontal Scaling     │
├────────────────────┼──────────────────────────┼──────────────────────────┤
│ How                │ Bigger machine            │ More machines            │
│ Upper limit        │ Hardware ceiling          │ Effectively unlimited    │
│ Cost curve         │ Exponential               │ Linear                   │
│ Fault tolerance    │ None (SPOF)               │ Built-in                 │
│ Code changes       │ None needed               │ Must be stateless        │
│ Deployment         │ Downtime likely           │ Rolling/zero-downtime    │
│ Best for           │ DBs, early stage, legacy  │ Web/API services         │
└────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

Topic 1 done. Solid on this? Then we move to **Topic 2 — Stateless Services**, which is the architectural prerequisite that makes horizontal scaling actually work in practice. Say the word.
