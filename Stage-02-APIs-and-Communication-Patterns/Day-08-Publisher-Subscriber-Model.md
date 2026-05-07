# Stage 2 — Topic 8: Publisher-Subscriber Model

## Theory

We have touched on pub/sub across the last two topics. Now we go deep — the architecture, the patterns, the internals, and the nuances that make pub/sub one of the most powerful and most misused patterns in distributed systems.

**The Publisher-Subscriber model** is a messaging pattern where:
- **Publishers** produce events without knowing who will consume them
- **Subscribers** consume events without knowing who produced them
- A **broker** sits in the middle, matching publishers to subscribers

The defining characteristic is **complete decoupling** — publishers and subscribers have zero knowledge of each other. This is stronger than just async communication — it is architectural ignorance by design.

```
Traditional direct communication:
  Order Service knows about → Notification Service
  Order Service knows about → Inventory Service
  Order Service knows about → Analytics Service
  Order Service knows about → Loyalty Service
  
  Adding a new consumer = modifying Order Service

Pub/Sub:
  Order Service knows about → Event Broker only
  Notification Service subscribes independently
  Inventory Service subscribes independently
  Analytics Service subscribes independently
  Loyalty Service subscribes independently
  
  Adding a new consumer = zero changes to Order Service
```

---

## Internals — How Pub/Sub Works

### The Three Participants

```
Publisher (Producer):
  - Knows the topic name
  - Publishes events to that topic
  - Has no knowledge of subscribers
  - Does not wait for acknowledgement from subscribers

Broker (Event Bus):
  - Accepts events from publishers
  - Stores events (temporarily or permanently)
  - Routes events to all registered subscribers
  - Manages subscriptions
  - Handles delivery guarantees

Subscriber (Consumer):
  - Registers interest in a topic
  - Receives events matching its subscription
  - Processes events independently
  - Has no knowledge of publisher identity
```

### The Full Flow

```
t=0:  Order Service publishes to topic "order.confirmed":
      {
        "eventId":   "evt-abc-123",
        "eventType": "ORDER_CONFIRMED",
        "orderId":   "o-789",
        "userId":    "u-123",
        "total":     1299.99,
        "timestamp": "2026-04-08T10:30:00Z"
      }

t=0:  Broker receives event, stores it, identifies all subscribers
      Subscribers to "order.confirmed":
        - notification-service
        - inventory-service
        - analytics-service
        - loyalty-service

t=0:  Broker delivers copy to each subscriber group

t=1ms:  Notification Service receives event → queues email job
t=3ms:  Inventory Service receives event → decrements stock
t=5ms:  Analytics Service receives event → records revenue
t=2ms:  Loyalty Service receives event → awards points

All four happen in parallel, independently, at their own pace.
```

---

## Pub/Sub Topologies — How Events Are Structured

### Topology 1 — Simple Fan-Out

```
One publisher → One topic → Multiple subscribers

order-service ──► [order.confirmed] ──► notification-service
                                    ──► inventory-service
                                    ──► analytics-service

Simple, clean. Every subscriber gets every event.
```

### Topology 2 — Filtered Subscription

Subscribers declare interest in only certain events — not all events on a topic:

```
order-service publishes ALL order events to one topic:
  ORDER_CREATED, ORDER_CONFIRMED, ORDER_SHIPPED, ORDER_DELIVERED, ORDER_CANCELLED

Subscribers filter by event type:

notification-service subscribes to:
  ORDER_CONFIRMED, ORDER_SHIPPED, ORDER_DELIVERED, ORDER_CANCELLED
  (needs to notify user at each stage)

inventory-service subscribes to:
  ORDER_CONFIRMED, ORDER_CANCELLED
  (confirmed = decrement stock, cancelled = restore stock)

analytics-service subscribes to:
  ORDER_CONFIRMED, ORDER_CANCELLED
  (revenue gained and refunded)

loyalty-service subscribes to:
  ORDER_CONFIRMED, ORDER_CANCELLED
  (points awarded and revoked)
```

```java
// Kafka — filter by event type in consumer
@KafkaListener(topics = "order-events", groupId = "inventory-service")
public void handleOrderEvent(OrderEvent event) {
    // Filter to only events this service cares about
    switch (event.getEventType()) {
        case ORDER_CONFIRMED -> inventoryService.decrementStock(event);
        case ORDER_CANCELLED -> inventoryService.restoreStock(event);
        default -> { /* ignore — not relevant to inventory */ }
    }
}
```

### Topology 3 — Topic Hierarchy

Organise events in a hierarchy — subscribers can subscribe broadly or narrowly:

```
Topic hierarchy (RabbitMQ topic exchange pattern):

order.confirmed          ← specific
order.cancelled          ← specific
order.shipped            ← specific
order.*                  ← all order events
payment.succeeded        ← specific
payment.failed           ← specific
payment.*                ← all payment events
#                        ← everything

Notification Service subscribes to: order.* and payment.*
Audit Service subscribes to:        #  (everything — for compliance log)
Analytics Service subscribes to:    order.confirmed and payment.succeeded
```

### Topology 4 — Event Aggregation

Multiple publishers, one subscriber aggregates:

```
order-service     ──►
payment-service   ──► [transaction-events] ──► audit-service
inventory-service ──►
user-service      ──►

Audit service subscribes to all services and builds a complete audit trail.
No individual service needs to know about auditing.
```

---

## Event Design — What Makes a Good Event

Event design is to pub/sub what API design is to REST. Bad event design creates downstream pain for every subscriber.

### Event Naming — Past Tense, Happened Facts

```
Bad (commands — tells someone what to do):
  SendEmail
  DecrementStock
  AwardPoints

Bad (present tense — ambiguous timing):
  OrderConfirming
  PaymentProcessing

Good (past tense — facts that happened):
  OrderConfirmed
  OrderCancelled
  PaymentSucceeded
  PaymentFailed
  StockDepleted
  UserRegistered
```

Events describe things that **already happened** — they are immutable facts. Commands tell someone what to do — those belong in queues, not pub/sub topics.

### Event Payload Design — Fat Events vs Thin Events

**Thin event (notification only):**
```json
{
  "eventType": "ORDER_CONFIRMED",
  "orderId": "o-789",
  "timestamp": "2026-04-08T10:30:00Z"
}
```
Subscriber receives this and must call Order Service to get the full order details.

Problems:
- Extra network call per subscriber per event
- Order Service gets hammered after every event
- If Order Service is down, subscribers cannot process the event

**Fat event (self-contained):**
```json
{
  "eventId":    "evt-abc-123",
  "eventType":  "ORDER_CONFIRMED",
  "timestamp":  "2026-04-08T10:30:00Z",
  "orderId":    "o-789",
  "userId":     "u-123",
  "userEmail":  "vidhan@example.com",
  "total":      1299.99,
  "items": [
    { "productId": "p-456", "name": "Nike Shoes", "qty": 2, "price": 649.99 }
  ],
  "shippingAddress": {
    "line1": "123 Main St",
    "city":  "Nagpur",
    "pin":   "440001"
  }
}
```

Subscriber has everything it needs — no additional calls required.

**Trade-offs:**
```
Thin events:
  ✅ Small payload size
  ✅ No data duplication
  ❌ Subscribers must make additional service calls
  ❌ Tight temporal coupling — subscriber needs publisher available

Fat events:
  ✅ Subscribers are self-sufficient — no additional calls
  ✅ Works even if publisher is temporarily down
  ✅ Natural for event sourcing and audit logs
  ❌ Larger payload size
  ❌ Data duplication across events
  ❌ If schema changes, all subscribers must update
```

**ShopSphere approach — fat events for domain events:**
Order-level events carry the full order snapshot. Subscribers process them independently. The slight payload increase is worth the resilience and simplicity.

### Event Versioning — Evolving Events Over Time

Events are harder to version than APIs because they may be stored for months and replayed:

**Strategy 1 — Version in event type:**
```json
{
  "eventType": "ORDER_CONFIRMED_V2",
  "version": 2,
  ...new fields...
}
```

**Strategy 2 — Version in schema (backward compatible additions only):**
```json
// V1 — original
{ "eventType": "ORDER_CONFIRMED", "orderId": "o-789", "total": 1299.99 }

// V2 — added optional field, backward compatible
{ "eventType": "ORDER_CONFIRMED", "orderId": "o-789", "total": 1299.99, "promoCode": "SAVE10" }

// Old subscribers ignore unknown fields — no code change needed
// New subscribers can use promoCode if present
```

**Strategy 3 — Consumer-driven contract testing:**
Each consumer publishes the event fields it depends on. The publisher runs tests ensuring its events satisfy all consumer contracts before deploying a change.

---

## Pub/Sub vs Observer Pattern — Know the Difference

The Observer pattern is pub/sub's in-process cousin — important for interviews:

```
Observer Pattern (in-process, same JVM):
  Subject maintains a list of Observer objects
  When state changes, Subject calls observer.update() directly
  Observers are registered explicitly with Subject
  All in one process — no network, no broker
  
  Example: Spring ApplicationEvent / ApplicationListener

Pub/Sub Pattern (distributed, across services):
  Publishers and subscribers know only the broker
  Broker handles routing, delivery, persistence
  Across network boundaries, separate processes
  Broker provides durability — messages survive crashes
  
  Example: Kafka, RabbitMQ, AWS SNS/SQS
```

```java
// Observer Pattern — Spring ApplicationEvent (in-process pub/sub)
// Publisher
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    public Order createOrder(CreateOrderRequest request) {
        Order order = orderRepository.save(buildOrder(request));
        
        // Publish in-process event — synchronous by default
        eventPublisher.publishEvent(new OrderCreatedEvent(this, order));
        
        return order;
    }
}

// Subscriber — same JVM
@Component
public class OrderCreatedEventListener {
    
    @EventListener
    @Async  // makes it asynchronous — runs in separate thread
    public void handleOrderCreated(OrderCreatedEvent event) {
        auditService.record(event.getOrder());
    }
}
```

**When to use Spring ApplicationEvent vs Kafka:**
```
Spring ApplicationEvent:
  Same service, in-process communication
  No persistence needed
  Decoupling within a monolith or within one service's components
  Low overhead — no network hop

Kafka/RabbitMQ:
  Cross-service communication
  Persistence needed — survive crashes
  Multiple independent services need the event
  High volume requiring durability and replay
```

---

## Backpressure in Pub/Sub — Handling Speed Mismatch

A critical operational concern — what happens when publishers produce faster than subscribers consume?

```
Producer:   10,000 events/second
Consumer:    2,000 events/second

After 1 minute:
  Unprocessed backlog: (10,000 - 2,000) × 60 = 480,000 messages

After 1 hour:
  Unprocessed backlog: 28,800,000 messages

Consumer is drowning — latency grows indefinitely
```

**Solutions:**

**1. Scale consumers horizontally:**
```
Add more consumer instances
Each gets a partition → parallel processing
5 consumer instances × 2,000 events/second = 10,000 events/second
Matches producer rate
```

**2. Consumer backpressure signalling:**
```
Kafka pull model naturally handles this:
  Consumer pulls messages at its own pace
  If consumer is slow — it just pulls less frequently
  Messages wait safely in the partition
  No overwhelming of consumer

RabbitMQ push model — configure prefetch:
  channel.basicQos(10) — broker sends max 10 unacked messages at once
  Consumer must ACK before broker sends more
  Prevents consumer from being flooded
```

**3. Alerting on consumer lag:**
```
Monitor consumer group lag:
  Lag = latest offset - consumer committed offset

Alert if lag exceeds threshold:
  lag > 10,000 → warning
  lag > 100,000 → critical — scale consumers or investigate

In ShopSphere: Prometheus + Grafana dashboards on Kafka consumer lag
```

---

## Fan-Out Problem — The Celebrity Problem in Pub/Sub

When one event generates an enormous number of downstream operations:

```
Scenario: A celebrity user with 10 million followers places an order
          (analogous to posting a tweet)

Event: USER_ACTIVITY published
Subscriber: Feed Service must update 10 million follower feeds
            → 10 million write operations from one event
            → Fan-out explosion
```

**Solutions:**

**Fan-out on write (precompute):**
```
When event published → immediately write to all 10 million feeds
Pros: Feed reads are instant — already computed
Cons: Write amplification — one event = 10 million writes
     Delayed for high-follower users during spike
```

**Fan-out on read (lazy compute):**
```
When event published → store once
When follower reads feed → compute their feed at read time
Pros: No write amplification
Cons: Feed read is slower — computed on demand
```

**Hybrid (real Twitter/Instagram approach):**
```
Regular users (< 1M followers): fan-out on write — precompute feeds
Celebrity users (> 1M followers): fan-out on read — compute at read time
                                  and cache aggressively

ShopSphere equivalent:
  Regular sellers: precompute product recommendations
  Top sellers with millions of followers: compute on read with Redis cache
```

---

## Real-World Example — ShopSphere Complete Pub/Sub Architecture

```java
// Order Service — publishes domain events
@Service
@Slf4j
public class OrderEventPublisher {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Autowired
    private OutboxRepository outboxRepository;

    @Transactional
    public void publishOrderConfirmed(Order order) {
        OrderConfirmedEvent event = OrderConfirmedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType("ORDER_CONFIRMED")
            .timestamp(Instant.now().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .userEmail(order.getUserEmail())
            .total(order.getTotal())
            .items(order.getItems().stream()
                .map(this::mapItem)
                .collect(Collectors.toList()))
            .build();

        // Outbox pattern — save atomically with order
        outboxRepository.save(OutboxEvent.from(event));
        
        log.info("OrderConfirmed event queued in outbox: {}", event.getOrderId());
    }
}

// ─────────────────────────────────────────────────────
// Notification Service — subscribes to order events
@Service
@Slf4j
public class OrderNotificationConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "notification-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(OrderEvent event,
                        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                        @Header(KafkaHeaders.OFFSET) long offset) {
        
        log.info("Received {} at offset {}", event.getEventType(), offset);
        
        // Idempotency check
        if (processedEvents.contains(event.getEventId())) {
            log.warn("Duplicate event {}, skipping", event.getEventId());
            return;
        }
        
        switch (event.getEventType()) {
            case "ORDER_CONFIRMED" -> {
                emailService.sendOrderConfirmation(
                    event.getUserEmail(),
                    event.getOrderId(),
                    event.getTotal()
                );
                smsService.sendConfirmation(event.getUserId(), event.getOrderId());
            }
            case "ORDER_SHIPPED" -> {
                emailService.sendShippingNotification(
                    event.getUserEmail(),
                    event.getTrackingNumber()
                );
            }
            case "ORDER_DELIVERED" -> {
                emailService.sendDeliveryConfirmation(event.getUserEmail());
                reviewRequestService.scheduleReviewRequest(
                    event.getUserId(), event.getOrderId()
                );
            }
        }
        
        processedEvents.add(event.getEventId()); // mark as processed
    }
}

// ─────────────────────────────────────────────────────
// Inventory Service — subscribes independently
@Service
public class InventoryEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void consume(OrderEvent event) {
        switch (event.getEventType()) {
            case "ORDER_CONFIRMED" ->
                inventoryService.decrementStock(event.getItems());
            case "ORDER_CANCELLED" ->
                inventoryService.restoreStock(event.getItems());
        }
    }
}
```

---

## Interview Q&A

**Q: What is the pub/sub pattern and how does it differ from direct service calls?**
In pub/sub, publishers emit events to a broker without knowing who consumes them, and subscribers consume events without knowing who produced them. In direct service calls, the caller knows the callee's address and calls it explicitly — tight coupling. Pub/sub provides complete decoupling — adding a new consumer requires zero changes to the publisher. It also provides temporal decoupling — publisher and subscriber do not need to be available simultaneously.

**Q: What is the difference between a fat event and a thin event?**
A thin event carries only identifiers — the subscriber must call back to the source service to get full details. This adds a network round trip per subscriber per event and creates temporal coupling — if the source service is down, subscribers cannot process the event. A fat event carries the complete snapshot of state at the time of the event — subscribers are self-sufficient and can process the event without any additional calls. Fat events are preferred for domain events because they improve resilience and simplicity at the cost of slightly larger payloads.

**Q: How do you handle schema evolution for events in a pub/sub system?**
Events are harder to evolve than APIs because they may be stored and replayed months later. Safe evolution means only adding optional fields — existing subscribers ignore unknown fields. Never remove or rename fields — mark them deprecated and maintain them. For breaking changes, introduce a new event type version and publish both old and new until all subscribers migrate. Consumer-driven contract testing helps catch breaking changes before deployment by validating that published events satisfy the fields each subscriber depends on.

**Q: What is backpressure in pub/sub and how do you handle it?**
Backpressure occurs when producers publish events faster than consumers can process them, causing the backlog to grow indefinitely. Kafka's pull model naturally handles this — consumers pull at their own pace and the backlog waits safely in partitions. For RabbitMQ's push model, configure prefetch limits so the broker only sends a bounded number of unacknowledged messages at once. Long-term solutions are scaling consumer instances horizontally to match production rate, and alerting on consumer group lag so backlogs are caught before they become critical.

**Q: How does pub/sub relate to the Observer pattern?**
The Observer pattern is pub/sub within a single process — a subject maintains a list of observer objects and calls them directly when state changes. Pub/sub is the distributed equivalent — publishers and subscribers are in separate processes or services, communicating through a broker that handles persistence, routing, and delivery guarantees. Spring ApplicationEvent is an in-process pub/sub mechanism suitable for decoupling within a single service. Kafka and RabbitMQ are distributed pub/sub mechanisms suitable for cross-service communication with durability requirements.

---

Say **"next"** when ready for Topic 8 — Event-Driven Architecture.
