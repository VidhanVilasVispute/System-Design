# Stage 8 — Microservices & Architecture Patterns

Great, let's go topic by topic, deep and structured. Starting with:

---

## Topic 1 — Microservice Architecture

### What is it?

A **Microservice Architecture** is a style where a large application is broken into **small, independently deployable services**, each responsible for a **single business capability**.

Contrast this with a **Monolith**, where everything lives in one deployable unit.

```
MONOLITH                          MICROSERVICES
─────────────────────────         ──────────────────────────────────────
│                       │         ┌──────────┐  ┌──────────┐  ┌───────┐
│  User + Order +       │         │  User    │  │  Order   │  │Product│
│  Product + Payment +  │   vs    │  Service │  │  Service │  │Service│
│  Notification all     │         └────┬─────┘  └────┬─────┘  └───┬───┘
│  in ONE codebase      │              │              │             │
│                       │         Each has own DB, own deploy, own scale
└───────────────────────┘
```

---

### Advantages

#### 1. Independent Deployment
Each service is deployed separately. You can push a fix to `Order Service` without touching `User Service` or `Payment Service`.

```
Team A ships Order Service v2.1 on Monday
Team B ships User Service v1.8 on Wednesday
→ Zero coordination needed between teams
```

#### 2. Independent Scalability
You scale only what's under load — not everything.

```
Black Friday scenario:
  Order Service    → scale to 20 pods  (high load)
  User Service     → stay at 3 pods    (normal load)
  Notification     → scale to 10 pods  (email surge)

Monolith: you'd scale the ENTIRE app even if only orders are bottlenecked.
```

#### 3. Technology Freedom (Polyglot)
Each service can use the best tool for its job.

```
Search Service     → Java + Elasticsearch
ML Recommendations → Python + TensorFlow
Real-time Chat     → Node.js + WebSocket
Payment Service    → Go (low latency, high throughput)
```

#### 4. Fault Isolation
A crash in one service doesn't bring down the whole system (if designed correctly with circuit breakers, timeouts, etc.)

```
Notification Service goes down
→ Orders still process
→ Users can still login
→ Notification just queues up and retries later
```

#### 5. Team Autonomy (Conway's Law alignment)
Small teams own small services. Faster iteration. Clear ownership.

---

### Disadvantages

#### 1. Operational Complexity
Instead of deploying 1 thing, you now deploy, monitor, and manage 20+ services.

```
Monolith: 1 app, 1 log file, 1 deployment
Microservices: 20 services × 3 replicas = 60 pods
              + 20 separate CI/CD pipelines
              + 20 health endpoints to monitor
              + distributed logs across all of them
```

#### 2. Network Latency & Failures
In a monolith, a method call is nanoseconds. In microservices, it's an HTTP/gRPC call over a network — which can fail, time out, or be slow.

```
User hits "Place Order"
→ Order Service calls Product Service (HTTP)  → might timeout
→ Order Service calls Payment Service (HTTP)  → might fail
→ Order Service calls Notification (RabbitMQ) → might be down

Monolith: all of this is just function calls in memory
```

#### 3. Distributed Transactions are Hard
What if Order is created, Payment succeeds, but Inventory update fails?

```
In a monolith: wrap everything in a DB transaction → rollback on failure. Easy.
In microservices: each service has its OWN DB → no shared transaction.
→ You need Saga Pattern or 2PC (both complex).
```

#### 4. Data Consistency (Eventual vs Strong)
Because each service has its own DB and communicates async (events), data may be **eventually consistent** — not immediately.

```
User updates email → User Service DB updated immediately
              ↓ publishes event
Order Service hears about it → 200ms later
→ For that 200ms, Order Service might email the old address
```

#### 5. Testing Complexity
Integration testing across 20 services is hard. You need contract testing, service virtualization, and complex test environments.

---

### ShopSphere Lens

Your ShopSphere is already a real microservices system. Look at how it maps:

```
ShopSphere Services:
┌─────────────────┬───────────────────────┬──────────────────────┐
│ Service         │ Independent DB        │ Why separate?        │
├─────────────────┼───────────────────────┼──────────────────────┤
│ user-service    │ PostgreSQL (users)    │ Auth + profile logic │
│ product-service │ PostgreSQL (products) │ Catalog management   │
│ order-service   │ PostgreSQL (orders)   │ Order lifecycle      │
│ search-service  │ Elasticsearch         │ Needs diff tech      │
│ review-service  │ PostgreSQL (reviews)  │ Independent scaling  │
│ notification    │ No state (events)     │ Fire-and-forget      │
└─────────────────┴───────────────────────┴──────────────────────┘
```

You've already seen the pain points: Redis auth issues affecting Gateway, Kafka events needing `afterCommit()` to avoid publishing before DB commit, Feign interceptors for internal header forwarding — all classic microservice complexity.

---

### Interview Questions on This Topic

**Q1. What's the difference between a monolith and microservices? When would you NOT choose microservices?**

> Microservices add operational overhead — distributed tracing, multiple deployments, network latency. For a small team or early-stage product, a **modular monolith** is often better. Microservices shine when you have multiple teams, independent scaling needs, or polyglot requirements.

**Q2. How do you handle a failure in one microservice without cascading?**

> **Circuit Breaker** pattern (Resilience4j). After N failures, the circuit "opens" — calls fail fast instead of waiting and piling up. Combined with fallbacks (return cached data or default response). Bulkhead pattern to isolate thread pools per downstream service.

**Q3. What is the hardest problem in microservices?**

> **Distributed transactions and data consistency**. There's no shared DB transaction. Solutions: Saga pattern (choreography or orchestration), outbox pattern for reliable event publishing, idempotent consumers for safe retries.

**Q4. You have 20 microservices. How do you debug a request that touched 10 of them?**

> Distributed tracing — each request gets a unique `correlationId`/`traceId` that propagates through all service calls. Tools like Jaeger or Zipkin aggregate the spans. This is Stage 8's later topic — Distributed Tracing.

---

Ready for **Topic 2 — Service Discovery** whenever you are. Just say the word.
