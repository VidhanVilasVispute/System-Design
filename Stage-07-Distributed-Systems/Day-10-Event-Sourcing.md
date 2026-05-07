# Stage 7 — Distributed Systems
## Topic 10: Event Sourcing

---

## The Problem With "Current State" Storage

Every database you've used so far stores **current state** — the latest value of each record.

```
orders table (traditional):
┌──────────┬─────────┬───────────┬────────────┬──────────────┐
│ order_id │ user_id │ product_id│   amount   │   status     │
├──────────┼─────────┼───────────┼────────────┼──────────────┤
│  uuid-1  │    7    │    42     │   $99.00   │  CONFIRMED   │
└──────────┴─────────┴───────────┴────────────┴──────────────┘
```

This tells you **what the order is right now**. It tells you nothing about:

```
❌ When was it placed?
❌ Was it ever cancelled and re-placed?
❌ Who changed the status and when?
❌ What was the original amount before a discount was applied?
❌ How many times did payment fail before succeeding?
❌ What sequence of events led to CONFIRMED?
```

You've lost all history. The past is gone.

---

## The Event Sourcing Idea

Instead of storing current state — **store every event that ever happened**.
Current state is derived by replaying those events.

```
order_events table (event sourced):
┌──────────┬───────────────────────┬────────────────────────────────┬──────────────────────┐
│ event_id │     aggregate_id      │         event_type             │       payload        │
├──────────┼───────────────────────┼────────────────────────────────┼──────────────────────┤
│    1     │  order-uuid-1         │  OrderPlaced                   │ {amount: 99, ...}    │
│    2     │  order-uuid-1         │  PaymentFailed                 │ {reason: "declined"} │
│    3     │  order-uuid-1         │  PaymentRetried                │ {attempt: 2}         │
│    4     │  order-uuid-1         │  PaymentProcessed              │ {txn_id: "tx-99"}    │
│    5     │  order-uuid-1         │  StockReserved                 │ {product_id: 42}     │
│    6     │  order-uuid-1         │  OrderConfirmed                │ {confirmed_at: ...}  │
└──────────┴───────────────────────┴────────────────────────────────┴──────────────────────┘
```

To get current state → replay events 1 through 6 in order.

```
Start:     { status: null }
+ event 1: { status: PLACED,     amount: 99 }
+ event 2: { status: PLACED,     paymentAttempts: 1, lastFailure: "declined" }
+ event 3: { status: PLACED,     paymentAttempts: 2 }
+ event 4: { status: PAID,       txnId: "tx-99" }
+ event 5: { status: PAID,       stockReserved: true }
+ event 6: { status: CONFIRMED,  confirmedAt: "2025-..." }

Current state: CONFIRMED ✅
Full history:  preserved ✅
```

---

## Core Concepts

### Aggregate

An **aggregate** is the domain object whose state you're tracking — Order, Product, Account.

```
Aggregate = Identity (UUID) + sequence of events applied to it

order-uuid-1's aggregate is the sum of all its events.
```

### Event

An event is an **immutable fact** — something that happened in the past.

```
Good event names (past tense, facts):
  OrderPlaced       ✅
  PaymentFailed     ✅
  StockReserved     ✅
  OrderConfirmed    ✅

Bad event names (commands, not facts):
  PlaceOrder        ❌  (that's a command)
  ProcessPayment    ❌  (that's a command)
```

Events are **immutable**. You never update or delete an event. The log only grows forward.

### Event Store

A database optimized for append-only event logs:

```
Requirements:
  Append-only writes          (no updates, no deletes)
  Read by aggregate ID        (give me all events for order-uuid-1)
  Ordered by sequence number  (events applied in correct order)
  Optimistic concurrency      (detect conflicting writes)
```

---

## Rebuilding State — The Replay Mechanism

```java
// Order aggregate — pure domain object
public class Order {

    private UUID orderId;
    private OrderStatus status;
    private BigDecimal amount;
    private int paymentAttempts;
    private String txnId;

    // Replay constructor — starts empty
    public Order() {}

    // Apply each event in sequence
    public void apply(DomainEvent event) {
        if (event instanceof OrderPlacedEvent e) {
            this.orderId = e.getOrderId();
            this.status  = OrderStatus.PLACED;
            this.amount  = e.getAmount();

        } else if (event instanceof PaymentFailedEvent e) {
            this.paymentAttempts++;

        } else if (event instanceof PaymentProcessedEvent e) {
            this.status = OrderStatus.PAID;
            this.txnId  = e.getTxnId();

        } else if (event instanceof StockReservedEvent) {
            // no state change needed, but event is still recorded

        } else if (event instanceof OrderConfirmedEvent) {
            this.status = OrderStatus.CONFIRMED;
        }
    }

    // Rebuild from event history
    public static Order rebuild(List<DomainEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }
}
```

```java
// Event store repository
@Repository
public class OrderEventStore {

    @Autowired JdbcTemplate jdbc;

    // Append new event
    public void append(UUID aggregateId, DomainEvent event, int expectedVersion) {
        try {
            jdbc.update("""
                INSERT INTO order_events
                  (event_id, aggregate_id, event_type, payload, sequence_number)
                VALUES (?, ?, ?, ?::jsonb, ?)
                """,
                UUID.randomUUID(),
                aggregateId,
                event.getClass().getSimpleName(),
                toJson(event),
                expectedVersion + 1    // optimistic concurrency check
            );
        } catch (DuplicateKeyException e) {
            // sequence_number unique constraint violated
            // another process wrote event at same version → conflict
            throw new ConcurrentModificationException(
                "Conflict on aggregate " + aggregateId);
        }
    }

    // Load all events for an aggregate
    public List<DomainEvent> load(UUID aggregateId) {
        return jdbc.query("""
            SELECT event_type, payload
            FROM order_events
            WHERE aggregate_id = ?
            ORDER BY sequence_number ASC
            """,
            (rs, row) -> deserialize(
                rs.getString("event_type"),
                rs.getString("payload")
            ),
            aggregateId
        );
    }
}
```

```java
// Service layer — load, act, save
@Service
public class OrderCommandService {

    @Autowired OrderEventStore eventStore;

    public void confirmOrder(UUID orderId) {
        // 1. Load all past events for this order
        List<DomainEvent> events = eventStore.load(orderId);

        // 2. Rebuild current state by replaying
        Order order = Order.rebuild(events);

        // 3. Validate business rule against current state
        if (order.getStatus() != OrderStatus.PAID) {
            throw new IllegalStateException("Can only confirm PAID orders");
        }

        // 4. Create new event
        OrderConfirmedEvent newEvent = new OrderConfirmedEvent(
            orderId, LocalDateTime.now()
        );

        // 5. Append to event store (optimistic concurrency)
        eventStore.append(orderId, newEvent, events.size());

        // 6. Optionally publish to Kafka for other services
        kafkaTemplate.send("order-events", newEvent);
    }
}
```

---

## The Performance Problem — And Snapshots

Replaying 6 events is fast. But what about:

```
A bank account that's been open for 10 years?
  → 50,000 transaction events

A product with 100,000 price change events?
  → Loading and replaying 100k events per request = unacceptable
```

**Snapshots** solve this:

```
Periodically serialize the current aggregate state as a snapshot.
On load: start from latest snapshot, replay only events AFTER it.

Without snapshot:    replay events 1 → 50,000     = slow ❌
With snapshot:       load snapshot at event 49,900
                     replay events 49,901 → 50,000  = fast ✅
```

```java
// Snapshot table
CREATE TABLE order_snapshots (
    aggregate_id   UUID         NOT NULL,
    version        INT          NOT NULL,  -- event sequence at snapshot time
    state          JSONB        NOT NULL,  -- serialized aggregate state
    created_at     TIMESTAMP    NOT NULL,
    PRIMARY KEY (aggregate_id, version)
);
```

```java
public Order loadWithSnapshot(UUID orderId) {
    // 1. Get latest snapshot
    Optional<Snapshot> snapshot = snapshotStore.getLatest(orderId);

    Order order;
    int fromVersion;

    if (snapshot.isPresent()) {
        // 2a. Deserialize snapshot as starting state
        order = fromJson(snapshot.get().getState(), Order.class);
        fromVersion = snapshot.get().getVersion();
    } else {
        // 2b. No snapshot — start from scratch
        order = new Order();
        fromVersion = 0;
    }

    // 3. Load only events AFTER snapshot
    List<DomainEvent> recentEvents =
        eventStore.loadFrom(orderId, fromVersion);

    // 4. Apply recent events on top of snapshot state
    recentEvents.forEach(order::apply);

    return order;
}
```

Snapshot strategy: take snapshot every N events (e.g., every 100 or 500 events).

---

## Projections — Turning Events Into Read Models

Event Sourcing naturally pairs with **CQRS** (next topic) because the event log is the write model, but you need efficient read models.

A **projection** listens to the event stream and builds a denormalized read-optimized view:

```
Event Stream:                          Projection (read model):
────────────────────────────────────────────────────────────────
OrderPlaced(orderId, userId, $99)
PaymentProcessed(orderId, txnId)   →   orders_summary table:
StockReserved(orderId, productId)      ┌──────────┬──────────┬──────────┐
OrderConfirmed(orderId)                │ order_id │ status   │ amount   │
                                       ├──────────┼──────────┼──────────┤
                                       │ uuid-1   │CONFIRMED │  $99     │
                                       └──────────┴──────────┴──────────┘
```

Multiple projections from the same event stream:

```
order_events stream
      │
      ├──▶ orders_summary        (order status dashboard)
      ├──▶ revenue_by_day        (analytics)
      ├──▶ user_order_history    (user profile page)
      ├──▶ fraud_detection_feed  (ML feature store)
      └──▶ shipping_queue        (fulfillment system)
```

Each projection is independent. A new one can be added by replaying the entire event history — retroactively building a view that didn't exist before.

---

## Temporal Queries — Event Sourcing's Superpower

Because all history is preserved, you can answer questions that are **impossible** with current-state storage:

```
"What did this order look like at 2:47 PM yesterday?"
→ Replay events up to that timestamp.

"How many orders were in PAYMENT_RETRYING state last Tuesday?"
→ Replay all events up to last Tuesday, count states.

"Show me every state this order has been in and when."
→ Apply events one by one, record state after each.

"What was the average payment retry rate last month?"
→ Count PaymentFailed events per order, correlate with PaymentProcessed.
```

With traditional storage: you'd need audit log tables, soft deletes, and complex history tracking added manually. With event sourcing: it's built in from day one.

---

## Event Schema Evolution — The Hard Part

Events are immutable and stored forever. But your domain changes. How do you evolve event schemas?

```
v1 event (stored 2 years ago):
  OrderPlacedEvent {
    orderId: UUID,
    amount:  BigDecimal
  }

v2 event (current):
  OrderPlacedEvent {
    orderId:    UUID,
    amount:     BigDecimal,
    currency:   String,      ← new field added
    discountId: UUID         ← new field added
  }
```

**Strategies:**

```
1. Upcasting (recommended):
   On load, transform old event format to new format
   before applying to aggregate.

   EventUpcaster: v1 OrderPlacedEvent
     → add currency = "INR" (sensible default)
     → add discountId = null
     → return v2 OrderPlacedEvent

2. Versioned events:
   Store event version number explicitly.
   event_type = "OrderPlacedEvent_v2"
   Apply different apply() logic per version.

3. Weak schema (Avro/Protobuf):
   Schema registry enforces forward/backward compatibility.
   New fields must have defaults.
   Fields can never be removed (deprecated instead).
```

---

## Event Sourcing in ShopSphere — Where It Fits

Not every service in ShopSphere needs event sourcing. Apply it surgically:

```
Service              ES Appropriate?   Reason
──────────────────────────────────────────────────────────────────────
order-service        ✅ YES            Full order lifecycle audit needed
                                       Payment retries, status changes
                                       Regulatory compliance

payment-service      ✅ YES            Financial audit trail mandatory
                                       Every charge, refund, retry must
                                       be permanently recorded

inventory-service    ⚠️ MAYBE         Stock movements are valuable history
                                       But high-frequency updates → snapshots
                                       required

user-service         ❌ NO            Current profile state sufficient
                                       GDPR "right to be forgotten" conflicts
                                       with immutable event log

product-service      ❌ NO            Catalog changes don't need full history
                                       Simple audit_log table is enough

notification-service ❌ NO            Append-only delivery log sufficient
                                       Full event sourcing is overkill
```

---

## Event Sourcing vs Traditional + Audit Log

You don't always need full event sourcing. A simpler audit log covers many cases:

```
Traditional + Audit Log:
──────────────────────────────────────────────────────────────
orders table          ← current state (fast reads)
order_audit_log table ← history of changes

order_audit_log:
  order_id | changed_by | old_status   | new_status  | changed_at
  uuid-1   | system     | PLACED       | CONFIRMED   | 2025-01-01

Pros: Simple, fast reads, existing tooling
Cons: Schema changes lose history, projection rebuilds are manual

Full Event Sourcing:
──────────────────────────────────────────────────────────────
order_events table ← ONLY source of truth

Pros: Complete history, projection rebuilds, temporal queries
Cons: Complex, read performance needs care, schema evolution is hard

Decision rule:
  Need to rebuild projections retroactively? → Event Sourcing
  Need full temporal queries? → Event Sourcing
  Just need "who changed what"? → Audit log is enough
```

---

## Event Store Options

```
Option                  Notes
──────────────────────────────────────────────────────────────────
EventStoreDB            Purpose-built event store, native projections
                        Best choice for pure event sourcing

PostgreSQL              Use append-only table with sequence_number
                        Simple, you already have it in ShopSphere
                        Good for getting started

Kafka                   Excellent event stream, topic per aggregate type
                        Log compaction for snapshots
                        But: no per-aggregate ordering guarantee

Axon Framework (Java)   Full ES + CQRS framework for Spring Boot
                        Handles event store, projections, sagas
                        Production-grade, widely used in Java ecosystem
```

---

## Event Sourcing + Outbox Pattern

One critical integration: when an event is stored, you also need to publish it to Kafka for other services.

The naive approach:

```java
eventStore.append(orderId, event);   // step 1
kafkaTemplate.send("events", event); // step 2 — what if this crashes?
```

If the service crashes between step 1 and step 2 → event stored but never published → other services never react.

The fix: **Outbox Pattern** (Topic 12). Store events AND outbox record in the same local transaction. A separate relay reads the outbox and publishes to Kafka.

```
order_events table  ──┐
                      ├── same local transaction ✅
outbox table        ──┘

outbox relay → Kafka → other services
```

Event sourcing and the outbox pattern are natural partners.

---

## Interview Angles 🎯

**Q: What is event sourcing and how does it differ from traditional state storage?**
> Traditional storage keeps only current state — the latest value of each field. Event sourcing stores every event that caused a state change, and current state is derived by replaying those events in order. You never lose history — every transition is permanently recorded as an immutable fact.

**Q: What is a projection in event sourcing?**
> A projection is a read model built by consuming the event stream. It transforms raw events into a denormalized view optimized for queries — like an orders summary table or a revenue dashboard. Multiple independent projections can be built from the same event stream, and any projection can be rebuilt from scratch by replaying the full event history.

**Q: How do snapshots work and when do you need them?**
> A snapshot is a periodic serialization of aggregate state at a known event sequence number. Instead of replaying all events from the beginning, you load the latest snapshot and replay only events after it. You need snapshots when aggregates accumulate hundreds or thousands of events — otherwise load times become unacceptable.

**Q: What's the hardest operational challenge with event sourcing?**
> Schema evolution. Events are immutable and stored forever, but your domain model changes. You need upcasting strategies to transform old event formats into current ones on load, or versioned event types with separate handling logic. Getting this wrong means you can't replay your history correctly — which defeats the entire purpose.

**Q: Why does event sourcing pair naturally with CQRS?**
> The event log is an append-only write model optimized for recording facts. But querying current state by replaying events is expensive. CQRS splits these concerns — the event store is the write side, and projections built from events are the read side. Each is optimized independently. This is why we cover CQRS right after event sourcing.

---

Say **next** for **Topic 11: CQRS** — separate read and write models, why it helps in high-read or complex domains 🚀
