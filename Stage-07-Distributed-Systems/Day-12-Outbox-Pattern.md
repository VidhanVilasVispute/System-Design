# Stage 7 — Distributed Systems
## Topic 12: Outbox Pattern

---

## The Problem — The Dual Write Problem

Every event-driven service in ShopSphere does two things on a write:

```java
@Transactional
public Order placeOrder(PlaceOrderCommand cmd) {
    Order order = Order.create(cmd);
    orderRepository.save(order);              // Step 1: write to DB
    kafkaTemplate.send("order-events", event); // Step 2: publish to Kafka
    return order;
}
```

This looks fine. It has a critical flaw.

```
These two operations are NOT atomic.
They touch two different systems.
No transaction spans both.

Crash scenarios:
──────────────────────────────────────────────────────────────
Crash after Step 1, before Step 2:
  → Order saved in DB ✅
  → Event never published ❌
  → inventory-service never reserves stock
  → payment-service never charges
  → Order stuck in PENDING forever — silent data loss 💥

Kafka down during Step 2:
  → Order saved in DB ✅
  → Kafka unavailable → publish fails ❌
  → Same result — event lost 💥

Step 2 succeeds but Step 1 failed (rolled back):
  → DB has no order ❌
  → Kafka has OrderCreated event ✅
  → Downstream services process a ghost order 💥
──────────────────────────────────────────────────────────────
```

This is the **dual write problem** — you can't atomically write to two different systems (DB + Kafka) without a distributed transaction.

And we just spent Topic 8 learning why distributed transactions are painful.

---

## The Outbox Pattern — The Solution

The key insight:

> **Don't write to Kafka directly. Write the event to your own database in the same transaction as your business data. Then a separate process reliably publishes from the database to Kafka.**

```
Before (broken):
  Business logic → DB write + Kafka publish (two systems, not atomic)

After (fixed):
  Business logic → DB write + outbox write (ONE transaction, one system)
  Outbox relay   → reads outbox → publishes to Kafka (separate concern)
```

```
┌─────────────────────────────────────────────────────────┐
│                    order-service                          │
│                                                           │
│  PlaceOrder ──▶ BEGIN TRANSACTION                        │
│                   INSERT INTO orders ...        ✅        │
│                   INSERT INTO outbox ...        ✅        │
│                 COMMIT  ←── atomic, single DB  ✅        │
│                                                           │
│  OutboxRelay (background)                                 │
│    SELECT unpublished FROM outbox                         │
│    → publish to Kafka ✅                                  │
│    → mark as published ✅                                 │
└─────────────────────────────────────────────────────────┘
```

Now:
- If service crashes after commit → outbox row exists → relay will publish it on recovery ✅
- If Kafka is down → outbox row waits → relay retries until Kafka recovers ✅
- If transaction rolls back → outbox row never inserted → no ghost event ✅

---

## The Outbox Table

```sql
-- Flyway migration: V4__create_outbox.sql
CREATE TABLE outbox (
    id              UUID            DEFAULT gen_random_uuid() PRIMARY KEY,
    aggregate_type  VARCHAR(100)    NOT NULL,  -- 'Order', 'Payment', etc.
    aggregate_id    UUID            NOT NULL,  -- the entity's ID
    event_type      VARCHAR(100)    NOT NULL,  -- 'OrderCreated', etc.
    payload         JSONB           NOT NULL,  -- full event data
    status          VARCHAR(20)     NOT NULL DEFAULT 'PENDING',
    created_at      TIMESTAMP       NOT NULL DEFAULT now(),
    published_at    TIMESTAMP,                 -- set when relay publishes
    retry_count     INT             NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_status_created
    ON outbox(status, created_at)
    WHERE status = 'PENDING';    -- partial index — only unpublished rows
```

---

## Writing to the Outbox

```java
// OrderService — one transaction, two inserts
@Service
public class OrderCommandService {

    @Autowired OrderRepository orderRepository;
    @Autowired OutboxRepository outboxRepository;
    @Autowired ObjectMapper objectMapper;

    @Transactional   // ← single DB transaction covers BOTH inserts
    public UUID placeOrder(PlaceOrderCommand cmd) {

        // Business write
        Order order = Order.builder()
            .id(UUID.randomUUID())
            .userId(cmd.getUserId())
            .productId(cmd.getProductId())
            .amount(cmd.getAmount())
            .status(OrderStatus.PENDING)
            .build();
        orderRepository.save(order);

        // Outbox write — same transaction
        OutboxEntry entry = OutboxEntry.builder()
            .aggregateType("Order")
            .aggregateId(order.getId())
            .eventType("OrderCreated")
            .payload(objectMapper.valueToTree(
                OrderCreatedEvent.from(order)
            ))
            .status(OutboxStatus.PENDING)
            .build();
        outboxRepository.save(entry);

        // NO kafkaTemplate.send() here
        return order.getId();
    }
}
```

Both inserts succeed or both roll back. Kafka is not touched. ✅

---

## The Outbox Relay — Two Approaches

### Approach 1: Polling Relay

A scheduled background thread polls the outbox table for unpublished entries:

```java
@Component
@Slf4j
public class OutboxPollingRelay {

    @Autowired OutboxRepository outboxRepo;
    @Autowired KafkaTemplate<String, Object> kafka;
    @Autowired ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 100)  // every 100ms
    @Transactional
    public void relay() {
        List<OutboxEntry> pending = outboxRepo
            .findTop100ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING);

        for (OutboxEntry entry : pending) {
            try {
                // determine Kafka topic from event type
                String topic = topicFor(entry.getEventType());

                // publish — Kafka send is sync here (get() blocks)
                kafka.send(topic, entry.getAggregateId().toString(),
                           entry.getPayload().toString())
                     .get(5, TimeUnit.SECONDS);

                // mark published
                entry.setStatus(OutboxStatus.PUBLISHED);
                entry.setPublishedAt(LocalDateTime.now());
                outboxRepo.save(entry);

            } catch (Exception e) {
                log.error("Failed to publish outbox entry {}", entry.getId(), e);
                entry.setRetryCount(entry.getRetryCount() + 1);
                if (entry.getRetryCount() >= 5) {
                    entry.setStatus(OutboxStatus.FAILED);  // alert needed
                }
                outboxRepo.save(entry);
            }
        }
    }

    private String topicFor(String eventType) {
        return switch (eventType) {
            case "OrderCreated"        -> "order-events";
            case "OrderStatusChanged"  -> "order-events";
            case "OrderCancelled"      -> "order-events";
            default -> throw new IllegalArgumentException(
                "Unknown event type: " + eventType);
        };
    }
}
```

**Problem with polling:** Under high load, 100ms polling creates latency. Under low load, it wastes DB queries.

Also — in a multi-replica deployment, multiple pods poll simultaneously:

```
Pod A polls → finds entry X → publishes to Kafka
Pod B polls → finds entry X → publishes to Kafka again!
→ Duplicate Kafka messages 💥
```

Fix: **pessimistic locking** on the SELECT:

```java
// Repository with SELECT FOR UPDATE SKIP LOCKED
@Query("""
    SELECT o FROM OutboxEntry o
    WHERE o.status = 'PENDING'
    ORDER BY o.createdAt ASC
    LIMIT 100
    FOR UPDATE SKIP LOCKED
    """,
    nativeQuery = true
)
List<OutboxEntry> findAndLockPending();
```

`SKIP LOCKED` — if another pod has locked a row, skip it instead of waiting.
Each pod processes a different batch. No duplicates. ✅

---

### Approach 2: Transactional Outbox via Debezium (CDC)

**Change Data Capture (CDC)** is a more elegant solution.

Instead of polling the outbox table from application code, a tool like **Debezium** reads the **PostgreSQL Write-Ahead Log (WAL)** directly and publishes changes to Kafka.

```
┌───────────────┐     WAL stream      ┌──────────────┐     ┌─────────┐
│  PostgreSQL   │ ──────────────────▶ │   Debezium   │ ──▶ │  Kafka  │
│               │   (binary log of    │  (CDC tool)  │     │         │
│  outbox table │    all DB changes)  └──────────────┘     └─────────┘
└───────────────┘
```

```
How it works:
────────────────────────────────────────────────────────────────
1. App writes to outbox table (inside business transaction)

2. PostgreSQL records the INSERT in its WAL
   (this happens for ALL writes — it's how PostgreSQL works)

3. Debezium reads the WAL continuously (like a replica)

4. Detects new INSERT in outbox table

5. Publishes event to Kafka automatically

6. No polling. No application-level relay code.
   No SELECT FOR UPDATE needed.
────────────────────────────────────────────────────────────────
```

**Why CDC is better:**

```
Polling Relay:                    CDC (Debezium):
──────────────────────────────────────────────────────────────
Adds DB load (SELECT every 100ms) Reads WAL — zero extra DB load
Latency = polling interval        Near real-time (milliseconds)
Duplicate handling needed         WAL is ordered, no duplicates
Application code to maintain      Infrastructure concern (Debezium)
Fails silently if relay crashes   Debezium persists its offset
```

**Debezium config for ShopSphere:**

```json
{
  "name": "shopsphere-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "shopsphere",
    "database.dbname": "orders",
    "table.include.list": "public.outbox",
    "transforms": "outbox",
    "transforms.outbox.type":
      "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.type": "event_type",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.table.field.event.key": "aggregate_id"
  }
}
```

Debezium's `EventRouter` transform routes each outbox row to a Kafka topic based on `aggregate_type` — zero application routing code.

---

## Idempotent Consumers — Handling Duplicates

Even with the outbox pattern, Kafka can deliver a message **more than once** — at-least-once delivery is the default.

```
Relay publishes event to Kafka ✅
Kafka ACKs receipt
Relay crashes before marking outbox entry as PUBLISHED
Relay restarts → finds entry still PENDING → publishes AGAIN

Result: two identical OrderCreated events on Kafka 💥
```

Every consumer must be **idempotent** — processing the same event twice produces the same result as processing it once.

```java
// inventory-service — idempotent consumer
@KafkaListener(topics = "order-events")
@Transactional
public void onOrderCreated(OrderCreatedEvent event) {

    // Check if already processed — idempotency key
    if (processedEventRepository.existsById(event.getEventId())) {
        log.info("Duplicate event {}, skipping", event.getEventId());
        return;   // already processed — safe to skip
    }

    // Process the event
    reserveStock(event.getProductId(), event.getOrderId());

    // Record that we've processed this event
    processedEventRepository.save(
        new ProcessedEvent(event.getEventId(), LocalDateTime.now())
    );
}
```

```sql
-- processed_events table — deduplication store
CREATE TABLE processed_events (
    event_id    UUID        PRIMARY KEY,
    processed_at TIMESTAMP  NOT NULL DEFAULT now()
);

-- Clean up old entries periodically (events older than 7 days)
-- A message arriving 7 days late is a bigger problem than dedup
```

---

## Outbox Pattern Variants

### Variant 1: Dedicated Outbox Table (shown above)
Most common. Clean separation. Easy to monitor.

### Variant 2: Event Table AS the Outbox
If you're doing Event Sourcing (Topic 10) — your event store IS the outbox. No separate table needed.

```
order_events table:
  → IS the source of truth for aggregate state
  → IS the outbox for publishing to Kafka
  → Debezium reads WAL of order_events → publishes to Kafka
```

One table, two purposes. Elegant.

### Variant 3: Domain Events in JPA Entity

Spring's `@DomainEvents` can collect events on the entity and publish after transaction commit:

```java
@Entity
public class Order extends AbstractAggregateRoot<Order> {

    public Order place() {
        // register event — published after @Transactional commits
        registerEvent(new OrderCreatedEvent(this.id, this.amount));
        return this;
    }
}

// Spring Data calls entity.domainEvents() after save() and publishes them
// via ApplicationEventPublisher — but this goes to in-memory listeners,
// NOT Kafka directly. Still needs outbox for true durability.
```

---

## Outbox in ShopSphere — Full Flow Diagram

```
order-service (place order):

  POST /orders
      │
      ▼
  @Transactional {
      INSERT INTO orders       (id, status=PENDING, ...)
      INSERT INTO outbox       (event_type=OrderCreated, payload=...)
  } COMMIT ──────────────────────────────────────── atomic ✅
      │
      │  (transaction committed — client gets 200 OK)
      │
      ▼
  OutboxRelay (background, 100ms polling / Debezium CDC)
      │
      ├── SELECT FROM outbox WHERE status=PENDING FOR UPDATE SKIP LOCKED
      │
      ├── kafka.send("order-events", OrderCreatedEvent)
      │
      └── UPDATE outbox SET status=PUBLISHED

  Kafka: "order-events" topic
      │
      ├──▶ inventory-service (idempotent) → reserve stock
      ├──▶ payment-service   (idempotent) → charge payment
      └──▶ notification-svc  (idempotent) → send confirmation email
```

---

## Monitoring the Outbox

In production, you must alert on:

```
1. PENDING entries older than N seconds
   → Relay is down or Kafka is unavailable
   → Alert: "Outbox lag > 30 seconds"

2. FAILED entries accumulating
   → Persistent Kafka publish failures
   → Alert: "Outbox has failed entries — manual intervention needed"

3. Outbox table size growing
   → Either relay is stopped or cleanup job is broken
   → Alert: "Outbox table > 10k rows"
```

```java
// Spring Boot Actuator custom metric for outbox lag
@Component
public class OutboxMetrics {

    @Autowired OutboxRepository outboxRepo;
    @Autowired MeterRegistry registry;

    @Scheduled(fixedDelay = 10_000)
    public void recordMetrics() {
        long pendingCount = outboxRepo.countByStatus(OutboxStatus.PENDING);
        long failedCount  = outboxRepo.countByStatus(OutboxStatus.FAILED);

        registry.gauge("outbox.pending.count", pendingCount);
        registry.gauge("outbox.failed.count", failedCount);
    }
}
```

---

## The Outbox Pattern — Summary Mental Model

```
Without Outbox:
  Write DB ──┐
             ├── NOT ATOMIC → crash between = data loss or ghost events
  Write Kafka┘

With Outbox:
  Write DB ──┐
             ├── ATOMIC (same DB transaction) → always consistent
  Write Outbox┘
       │
       │ (separate concern, retryable, idempotent)
       ▼
  Publish Kafka ← can fail and retry safely, outbox row is still there
```

The outbox pattern converts an **unreliable dual write** into a **reliable single write + retryable async publish**.

---

## Interview Angles 🎯

**Q: What is the dual write problem?**
> When a service writes to its database and publishes to a message broker in the same operation, these are two separate systems with no shared transaction. A crash between them leaves the system in an inconsistent state — either data was saved but the event wasn't published, or the event was published for data that was rolled back. The outbox pattern solves this.

**Q: How does the outbox pattern solve the dual write problem?**
> Instead of writing to Kafka directly, the service writes the event to an outbox table in the same database transaction as the business data. A separate relay process then reads the outbox and publishes to Kafka. The business write and event record are atomic — same DB transaction. The Kafka publish is a separate, retryable operation with no atomicity requirement.

**Q: What is CDC and how does Debezium use it for the outbox pattern?**
> Change Data Capture reads the database's write-ahead log — the binary record of every change — rather than querying the DB. Debezium acts like a replica, streaming changes from the WAL in real time. For the outbox pattern, Debezium detects inserts to the outbox table from the WAL and publishes them to Kafka automatically — no polling, no application relay code, near zero additional DB load.

**Q: Why must outbox consumers be idempotent?**
> The relay uses at-least-once delivery — if it crashes after publishing to Kafka but before marking the outbox entry as published, it will re-publish on recovery. Consumers must handle duplicates safely. The standard approach is storing processed event IDs in a deduplication table and skipping events whose IDs already exist.

**Q: How would you monitor the outbox in production?**
> Track three metrics: pending entry count with age (entries older than N seconds indicate relay lag or Kafka unavailability), failed entry count (persistent publish failures needing intervention), and overall table growth rate. Alert on each. The outbox table should stay near-empty in normal operation — accumulation is a signal of infrastructure problems.

---

Say **next** for **Topic 13: Write-Ahead Log (WAL)** — how databases guarantee durability before confirming writes, and why it's the foundation of everything we've covered 🚀
