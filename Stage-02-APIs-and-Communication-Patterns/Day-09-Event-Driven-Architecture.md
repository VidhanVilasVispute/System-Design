# Stage 2 — Topic 9: Event-Driven Architecture

## Theory

Event-Driven Architecture (EDA) is not just pub/sub with a fancier name. It is a complete architectural philosophy — a way of designing entire systems where **events are the primary mechanism of communication, state change, and integration**.

In a traditional request-driven system, services ask each other for things. In an event-driven system, services **announce what happened** and other services react. The difference sounds subtle but it fundamentally changes how you think about coupling, state, and system behaviour.

**The core philosophy shift:**

```
Request-Driven Thinking:
  "Order Service, please send an email to user u-123"
  → Order Service knows about Notification Service
  → Order Service is responsible for triggering email
  → Adding SMS requires modifying Order Service

Event-Driven Thinking:
  "An order was confirmed — here is the fact"
  → Order Service knows nothing about what happens next
  → Notification Service reacts and sends email
  → SMS Service reacts and sends SMS
  → Adding push notifications requires zero changes to Order Service
```

**The three pillars of EDA:**
- **Events** — immutable facts about things that happened
- **Event producers** — services that detect and publish state changes
- **Event consumers** — services that react to events and take action

---

## What Is an Event — Precisely

An event is a **record of something that happened in the past**. Three properties make something a true event:

**1. Immutable — it cannot be changed:**
```
OrderConfirmed at 10:30:00 is a permanent fact.
You cannot un-confirm an order.
If you cancel it later, that is a NEW event — OrderCancelled at 11:45:00.
The original event still exists.
```

**2. Past tense — it already happened:**
```
Not "ConfirmOrder" (command — future intent)
Not "OrderConfirming" (present — in progress)
Yes "OrderConfirmed" (past — it happened)
```

**3. Fact — not an instruction:**
```
An event does not tell subscribers what to do.
It states what happened.
Subscribers decide independently how to react.

OrderConfirmed → 
  Notification Service decides: send email
  Inventory Service decides: decrement stock
  Loyalty Service decides: award points
  
Each subscriber owns its own reaction logic.
```

---

## Event Types — Three Distinct Categories

Not all events are the same. Knowing these categories is an interview differentiator:

### Category 1 — Domain Events

Things that happened in your business domain. The most common type.

```
OrderConfirmed
PaymentSucceeded
ProductOutOfStock
UserRegistered
ReviewSubmitted
ShipmentDispatched
```

These are the events your business stakeholders would recognise. They represent meaningful state transitions in your core domain.

### Category 2 — Integration Events

Events that cross service boundaries — used specifically for inter-service communication.

```
// Internal domain event (within Order Service):
OrderPlaced { orderId, items, userId, internalData... }

// Integration event (published to other services):
OrderConfirmedIntegrationEvent { orderId, userId, total, items }
// ↑ Contains only what other services need — no internal implementation details
```

The distinction matters because domain events may contain internal implementation details that should not leak to other services. Integration events are the public API of your service's event stream — they must be designed as carefully as REST API contracts.

### Category 3 — System/Infrastructure Events

Technical events not tied to business logic:

```
ServiceStarted
HealthCheckFailed
ConfigurationChanged
DeploymentCompleted
CircuitBreakerOpened
```

These drive operational automation — auto-scaling, alerting, self-healing systems.

---

## EDA Patterns — The Four Core Patterns

### Pattern 1 — Event Notification

The simplest form. A service publishes an event to notify others that something happened. No payload details — just the notification.

```
UserService publishes: UserEmailChanged { userId: "u-123" }

Subscribers:
  NotificationService → "I should send a verification email to new address"
  AuthService         → "I should invalidate existing sessions"
  AuditService        → "I should log this change"
```

**Characteristics:**
- Minimal payload — just enough to identify what changed
- Subscribers must query for details if needed
- Very loose coupling — publisher exposes nothing
- Risk: subscriber must call back, temporal coupling reappears

**When to use:** When you want maximum decoupling and subscribers have easy access to query the source for details.

---

### Pattern 2 — Event-Carried State Transfer

Events carry the full state snapshot — subscribers update their own local copy of the data.

```
ProductService publishes:
ProductUpdated {
  productId:   "p-456",
  name:        "Nike Air Max",
  price:       129.99,
  description: "...",
  category:    "shoes",
  inventory:   45,
  updatedAt:   "2026-04-08T10:30:00Z"
}

SearchService consumes this event and:
  → Updates its own Elasticsearch index
  → No need to call ProductService ever
  → Has a local copy of all product data it needs

RecommendationService consumes this event and:
  → Updates its own product feature store
  → No need to call ProductService ever
```

**Characteristics:**
- Fat events — full state in the payload
- Each subscriber maintains its own read model
- Temporal decoupling — subscriber works even if publisher is down
- Data duplication — same data lives in multiple services
- Eventually consistent — subscriber's copy lags slightly

**When to use:** When subscribers need to query data frequently and you want them fully autonomous. This is the basis of the CQRS read model pattern.

---

### Pattern 3 — Event Sourcing

Instead of storing current state, store the **complete sequence of events** that led to the current state. Current state is derived by replaying the event log.

```
Traditional state storage (store current state):
  orders table:
    id: o-789, status: DELIVERED, total: 1299.99

Event Sourcing (store all events):
  order_events table:
    1: OrderCreated     { orderId: o-789, userId: u-123, items: [...] }
    2: PaymentProcessed { orderId: o-789, amount: 1299.99 }
    3: OrderConfirmed   { orderId: o-789 }
    4: OrderShipped     { orderId: o-789, trackingNo: "TRK-001" }
    5: OrderDelivered   { orderId: o-789, deliveredAt: "2026-04-09" }

Current state = replay events 1 through 5
```

**The power of event sourcing:**

```
1. Complete audit trail — every state change is recorded forever
   "Who changed this order and when?" — trivial to answer

2. Time travel — reconstruct state at any point in time
   "What was the order status at 10:30am?" — replay events up to that timestamp

3. Replay for new features — add a new read model retroactively
   New analytics service added today can replay all historical events
   and build a complete picture from day one

4. Debugging — reproduce any bug by replaying events that led to it

5. Event-driven natural fit — events ARE the storage format
```

**The costs:**

```
1. Complexity — querying current state requires replaying events
   Solved by: snapshots (periodic materialised state)
              projections (precomputed read models)

2. Schema evolution — old events must be readable forever
   Cannot change event structure without migration strategy

3. Large event stores — years of events accumulate
   Solved by: snapshots, archival, compaction
```

**Snapshot optimisation:**
```
Without snapshots:
  Replay 50,000 events to get current order state = slow

With snapshots:
  Store snapshot at event 49,000: { status: SHIPPED, ... }
  Replay only events 49,001 to 50,000 = fast

Snapshot strategy:
  Take snapshot every N events, or on every state change
  Keep last K snapshots for recovery
```

---

### Pattern 4 — CQRS (Command Query Responsibility Segregation)

Separate the **write model** (how you change state) from the **read model** (how you query state).

```
Traditional unified model:
  Same database handles all writes AND reads
  Complex queries slow down writes
  Write schema optimised for integrity, not for read performance

CQRS:
  Write side:  Command handlers → Write DB (optimised for writes, integrity)
  Read side:   Query handlers  → Read DB  (optimised for reads, denormalised)

  Events bridge the two:
  Write DB change → event published → Read DB updated via event handler
```

**ShopSphere CQRS example — Order read model:**

```
Write side (PostgreSQL — normalised, transactional):
  orders table:    { id, userId, status, total, createdAt }
  order_items:     { id, orderId, productId, qty, price }
  
  Optimised for: ACID transactions, data integrity

Read side (Elasticsearch or Redis — denormalised, fast):
  order_summary: {
    orderId:      "o-789",
    userId:       "u-123",
    userName:     "Vidhan",         ← denormalised from users table
    status:       "CONFIRMED",
    total:        1299.99,
    itemCount:    3,
    productNames: ["Nike Shoes", "..."],  ← denormalised from products
    createdAt:    "2026-04-08T10:30:00Z"
  }
  
  Optimised for: fast reads, full-text search, filtering

Event bridge:
  OrderConfirmed → event handler → updates order_summary in Elasticsearch
  UserNameChanged → event handler → updates all order_summaries for that user
```

**Why CQRS matters:**

```
Without CQRS — the impedance mismatch problem:
  Admin dashboard query:
    SELECT o.*, u.name, u.email, COUNT(oi.id) as itemCount,
           GROUP_CONCAT(p.name) as productNames
    FROM orders o
    JOIN users u ON o.userId = u.id
    JOIN order_items oi ON o.id = oi.orderId
    JOIN products p ON oi.productId = p.id
    WHERE o.status = 'CONFIRMED'
    GROUP BY o.id
    
  → Complex JOIN on write DB → slows down order creation transactions
  → Adding an index for reads hurts write performance

With CQRS:
  Admin dashboard query:
    GET /api/admin/orders?status=CONFIRMED
    → Elasticsearch query on pre-built read model
    → No JOINs, no impact on write DB, millisecond response
```

---

## EDA vs Request-Driven — When Each Fits

```
Request-Driven is better when:
  → You need an immediate response to continue flow
  → Strong consistency is required right now
  → The interaction is simple and bounded
  → Operation must be confirmed before proceeding
  
  Examples:
    Payment processing (must know if it succeeded)
    Login authentication (must verify credentials now)
    Inventory check before purchase (must know if in stock)

Event-Driven is better when:
  → Multiple services react to the same trigger
  → Reaction can happen after the fact
  → You want to add new consumers without modifying producers
  → Services should be independently deployable and scalable
  → You need a complete audit trail
  → Eventual consistency is acceptable
  
  Examples:
    Post-order processing (email, loyalty, analytics)
    Search index updates
    Recommendation model updates
    Audit logging
    Cross-service data synchronisation
```

---

## EDA Pitfalls — What Goes Wrong

### Pitfall 1 — Choreography Hell

```
With choreography (pure EDA — services react to each other's events):

Order Confirmed →
  Inventory Service decrements stock →
    StockUpdated event →
      WarehouseService picks item →
        ItemPicked event →
          ShippingService creates label →
            LabelCreated event →
              NotificationService sends email →
                ...

A chain of 8 events across 8 services.
If step 5 fails, which services know?
How do you debug what happened?
How do you compensate if shipping fails after inventory was decremented?
```

**Solution — Saga Pattern (we cover this in Stage 7):**
Use an orchestrator for complex multi-step flows — a single process that coordinates the steps and handles failures explicitly.

### Pitfall 2 — Eventual Consistency UX Problems

```
User places order → immediately checks order status
System: "Order not found" (read model not yet updated)
User: confused and frustrated

Solution:
  Return the order data in the creation response
  Do not redirect users to a page that queries the read model immediately
  Or use synchronous read after write — read from write model briefly after creation
```

### Pitfall 3 — Event Ordering Assumptions

```
Publisher sends:
  Event 1: OrderConfirmed
  Event 2: OrderShipped

Consumer receives (due to partition reassignment):
  Event 2: OrderShipped  ← out of order!
  Event 1: OrderConfirmed

Inventory was decremented for a shipped order that the consumer
thinks is still pending — inconsistent state.

Solution:
  Use event sequence numbers
  Consumers detect and handle out-of-order events
  Use Kafka partition keys (orderId) to guarantee order within a partition
```

### Pitfall 4 — Missing Events

```
Service deployed mid-stream — missed events during downtime.
Service added later — no access to historical events.

Solution:
  Kafka with sufficient retention period (7–30 days)
  Event sourcing — replay from the beginning
  Consumer group offsets — rewind to reprocess
```

---

## Real-World Example — ShopSphere Full EDA Flow

```java
// ── WRITE SIDE ──────────────────────────────────────────────────

// Order placed — triggers the entire EDA flow
@Service
public class OrderCommandHandler {

    @Transactional
    public Order handle(PlaceOrderCommand command) {
        
        // Synchronous critical path
        validateOrder(command);
        reserveInventory(command.getItems());      // gRPC — sync
        PaymentResult payment = processPayment();  // gRPC — sync
        
        // Build order aggregate
        Order order = Order.create(command, payment);
        orderRepository.save(order);
        
        // Publish domain event — stored in outbox atomically
        eventStore.append(OrderConfirmedEvent.from(order));
        
        return order;
        // User gets response here — all async work happens below
    }
}

// ── EVENT STORE / OUTBOX PROCESSOR ──────────────────────────────

@Scheduled(fixedDelay = 500)
public void processOutbox() {
    outboxRepository.findUnpublished().forEach(event -> {
        kafkaTemplate.send("order-events",
            event.getAggregateId(),  // partition key = orderId
            event.getPayload());
        outboxRepository.markPublished(event.getId());
    });
}

// ── READ SIDE — PROJECTIONS ──────────────────────────────────────

// Notification projection
@KafkaListener(topics = "order-events", groupId = "notification-projection")
public void project(OrderEvent event) {
    if ("ORDER_CONFIRMED".equals(event.getType())) {
        emailService.sendConfirmation(event.getUserEmail(), event.getOrderId());
    }
}

// Search projection — maintains Elasticsearch read model
@KafkaListener(topics = "order-events", groupId = "search-projection")
public void project(OrderEvent event) {
    OrderDocument doc = OrderDocument.builder()
        .orderId(event.getOrderId())
        .userId(event.getUserId())
        .status(event.getStatus())
        .total(event.getTotal())
        .productNames(extractProductNames(event.getItems()))
        .build();
    
    elasticsearchClient.index(i -> i
        .index("orders")
        .id(event.getOrderId())
        .document(doc));
}

// Analytics projection — maintains time-series read model
@KafkaListener(topics = "order-events", groupId = "analytics-projection")
public void project(OrderEvent event) {
    if ("ORDER_CONFIRMED".equals(event.getType())) {
        revenueStore.record(
            event.getTimestamp(),
            event.getTotal(),
            event.getItems()
        );
    }
}
```

---

## Interview Q&A

**Q: What is Event-Driven Architecture and how is it different from pub/sub?**
Pub/sub is a messaging pattern — a mechanism for decoupled message delivery. Event-Driven Architecture is a broader architectural philosophy where events are the primary means of communication, state change, and integration across the entire system. EDA uses pub/sub as its communication mechanism but goes further — events are first-class citizens, services are designed to react rather than be called, and the event log can become the source of truth for system state. Pub/sub is a tool; EDA is an architectural style.

**Q: What is the difference between event notification and event-carried state transfer?**
Event notification carries minimal data — just enough to signal that something changed. Subscribers must call back to the source for details, which reintroduces temporal coupling. Event-carried state transfer includes the full data snapshot in the event payload — subscribers update their own local copy and never need to call the source service. ECST makes subscribers fully autonomous and resilient to source service downtime at the cost of larger payloads and data duplication.

**Q: What is Event Sourcing and when would you use it?**
Event sourcing stores the complete history of events that led to the current state rather than just the current state snapshot. Current state is derived by replaying the event log. Use it when you need a complete audit trail — financial systems, compliance-heavy domains — when you need time travel to reconstruct past state, when you want to add new read models retroactively by replaying historical events, or when your domain is naturally event-oriented. The costs are complexity in querying current state — solved with snapshots and projections — and the need to maintain backward-compatible event schemas indefinitely.

**Q: What is CQRS and why is it used with Event-Driven Architecture?**
CQRS separates the write model from the read model. The write side handles commands and maintains a normalised, transactional database optimised for integrity. The read side maintains denormalised, query-optimised projections. Events bridge the two — when the write side changes state, it publishes an event, and read-side projectors update their models. EDA makes CQRS practical because events naturally synchronise write and read sides asynchronously. The benefit is that complex read queries never impact write performance — each side is optimised independently.

**Q: What is choreography vs orchestration in EDA and what are the trade-offs?**
Choreography is pure EDA — each service reacts to events and publishes its own events, with no central coordinator. It is highly decoupled but complex multi-step flows become hard to debug and reason about — you must trace a chain of events across many services to understand what happened. Orchestration uses a central saga orchestrator that explicitly coordinates each step, knows the full flow, and handles failures and compensations. Choreography is better for simple independent reactions. Orchestration is better for complex multi-step business processes where you need explicit failure handling and visibility into the overall flow.

---

## Stage 2 Progress

```
✅ Topic 1  — What is an API
✅ Topic 2  — REST API Design (Deep Dive)
✅ Topic 3  — gRPC (Deep Dive)
✅ Topic 4  — GraphQL (Deep Dive)
✅ Topic 5  — Synchronous vs Asynchronous Communication
✅ Topic 6  — Message-Based Communication
✅ Topic 7  — Publisher-Subscriber Model
✅ Topic 8  — Event-Driven Architecture
⬜ Topic 9  — Authentication vs Authorization
⬜ Topic 10 — JWT & OAuth2
⬜ Topic 11 — API Gateway
```

---

Say **"next"** when ready for Topic 9 — Authentication vs Authorization.
