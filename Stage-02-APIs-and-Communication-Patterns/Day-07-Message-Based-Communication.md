# Stage 2 — Topic 7: Message-Based Communication

## Theory

In the previous topic we established *why* asynchronous communication exists. Now we go deep on *how* it works — the infrastructure, patterns, and internals of message-based communication.

**Message-based communication** means services talk to each other by sending messages through an intermediary — a **message broker**. Services never talk directly. They produce messages to the broker and consume messages from the broker.

```
Direct communication (no broker):
  Service A ──────────────────────► Service B
  Tight coupling — A must know B's address, B must be up

Message-based communication:
  Service A ──► Broker ──► Service B
  Loose coupling — A only knows the broker, not B
```

The broker is the backbone of async architecture. It stores messages, routes them, manages delivery guarantees, and handles the mismatch between producer speed and consumer speed.

---

## Core Concepts — The Building Blocks

### Messages

A message is the unit of communication. It has two parts:

```
Message:
  Header (metadata):
    messageId:     "msg-abc-123"        ← unique ID
    correlationId: "order-o-789"        ← links to originating operation
    timestamp:     "2026-04-08T10:30Z"
    contentType:   "application/json"
    source:        "order-service"
    retryCount:    0
    
  Body (payload):
    {
      "eventType": "ORDER_CONFIRMED",
      "orderId":   "o-789",
      "userId":    "u-123",
      "total":     1299.99,
      "items":     [...]
    }
```

The header is for infrastructure — routing, tracing, deduplication. The body is for business logic — what actually happened.

### Queues vs Topics — The Fundamental Split

This is the most important distinction in message-based communication:

**Queue — point-to-point:**
```
Producer ──► [Queue] ──► ONE Consumer

Properties:
  - Each message is delivered to exactly ONE consumer
  - Competing consumers share the workload
  - Message is deleted after successful consumption
  - Used for task distribution — work that needs to happen once

Example:
  Order Service ──► [email-queue] ──► Notification Service instance 1
                                  or  Notification Service instance 2
                                  or  Notification Service instance 3
  (whichever is free processes it — load balanced automatically)
```

**Topic — publish/subscribe:**
```
Producer ──► [Topic] ──► Consumer Group A (all instances get it)
                     ──► Consumer Group B (all instances get it)
                     ──► Consumer Group C (all instances get it)

Properties:
  - Each message is delivered to ALL subscriber groups
  - Within a group, one instance processes each message
  - Message is retained for a configurable period (not deleted on consumption)
  - Used for event broadcasting — things that happened that many services care about

Example:
  Order Service ──► [order-confirmed topic]
                        ──► Notification Service consumer group  → sends email
                        ──► Inventory Service consumer group     → decrements stock
                        ──► Analytics Service consumer group     → records revenue
                        ──► Loyalty Service consumer group       → awards points
```

**Queue = work distribution. Topic = event broadcasting.** Know this cold.

---

## The Two Dominant Brokers — RabbitMQ vs Kafka

ShopSphere uses both. Understanding when to use each is a real interview question.

### RabbitMQ — The Traditional Message Broker

RabbitMQ is a **message broker** — its job is to route messages and ensure delivery. Once a message is consumed and acknowledged, it is gone.

**Architecture:**
```
Producer
   │
   ▼
Exchange  ←── routing rules (bindings)
   │
   ├──► Queue A  ──► Consumer Group A
   ├──► Queue B  ──► Consumer Group B
   └──► Queue C  ──► Consumer Group C
```

**Exchange types — how messages are routed:**

```
Direct Exchange:
  Route by exact routing key
  order.confirmed → notification-queue (exact match)
  order.cancelled → refund-queue       (exact match)

Topic Exchange:
  Route by pattern matching
  order.*   → all order events queue
  *.failed  → all failure events queue
  #         → everything queue (# matches zero or more words)

Fanout Exchange:
  Broadcast to ALL bound queues regardless of routing key
  Notification to every subscriber — no filtering

Headers Exchange:
  Route by message header attributes
  contentType=invoice AND region=india → india-invoice-queue
```

**RabbitMQ delivery guarantees:**
```
At most once:  message delivered 0 or 1 times (fire and forget, can lose messages)
At least once: message delivered 1 or more times (duplicates possible, no loss)
Exactly once:  not natively supported — requires idempotent consumers
```

**Acknowledgement model:**
```java
// Manual acknowledgement — RabbitMQ holds message until ACK received
@RabbitListener(queues = "order.notifications")
public void handleOrderConfirmed(OrderConfirmedEvent event, Channel channel,
                                  @Header(AmqpHeaders.DELIVERY_TAG) long tag)
        throws IOException {
    try {
        emailService.send(event.getUserId(), event.getOrderId());
        channel.basicAck(tag, false);    // ✅ success — message deleted from queue
    } catch (Exception e) {
        channel.basicNack(tag, false, true);  // ❌ failure — requeue for retry
    }
}
```

If the consumer crashes before ACKing, RabbitMQ redelivers to another consumer — no message loss.

**Dead Letter Queue (DLQ):**
```
Message fails processing → retried N times → sent to Dead Letter Queue

DLQ is a queue for messages that could not be processed.
Operations team inspects, fixes, and republishes from DLQ.
Without DLQ, failed messages are lost or block the queue forever.
```

```yaml
# Spring Boot RabbitMQ DLQ configuration
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000    # 1 second
          multiplier: 2             # exponential backoff
          max-interval: 10000       # max 10 seconds
```

---

### Kafka — The Distributed Event Log

Kafka is fundamentally different from RabbitMQ. It is not a message broker — it is a **distributed append-only log**. Messages are not deleted after consumption. The log is the source of truth.

**Architecture:**
```
Producers
   │
   ▼
Topic: order-events
  Partition 0: [msg1][msg3][msg5][msg7]...  ← append only, immutable
  Partition 1: [msg2][msg4][msg6][msg8]...  ← append only, immutable
  Partition 2: [msg9][msg10][msg11]...      ← append only, immutable

Consumer Group A (Notification):
  Instance 1 → reads Partition 0
  Instance 2 → reads Partition 1
  Instance 3 → reads Partition 2

Consumer Group B (Analytics):
  Instance 1 → reads Partition 0  ← completely independent offset
  Instance 2 → reads Partition 1  ← does not affect Group A's progress
```

**The Log — Kafka's core concept:**
```
Offset:  0      1      2      3      4      5      6
         │      │      │      │      │      │      │
         ▼      ▼      ▼      ▼      ▼      ▼      ▼
        [e1]  [e2]  [e3]  [e4]  [e5]  [e6]  [e7]

Consumer Group A offset: 6 (has read up to e6)
Consumer Group B offset: 4 (is behind — still processing)
Consumer Group C offset: 7 (fully caught up)

Messages are NOT deleted when consumed.
Retention period determines when they are deleted (e.g. 7 days).
```

**Why this matters:**
- Consumer can re-read messages — replay from offset 0 to rebuild state
- New consumer group can start from the beginning and process all historical events
- Multiple consumer groups are completely independent — one being slow does not affect others
- Event sourcing is natural — the log IS the history

**Partitioning — Kafka's scalability mechanism:**
```
Partition key determines which partition a message goes to:
  key = userId → all events for user U always go to same partition

Why this matters for ordering:
  Kafka only guarantees ordering WITHIN a partition
  Events for the same order must go to the same partition
  → use orderId as partition key

Parallelism:
  10 partitions → up to 10 consumers in a group process in parallel
  Want more parallelism? Add more partitions.
  Each partition is an independent ordered log.
```

**Kafka producer configuration — the throughput vs latency trade-off:**
```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        
        // Durability
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        // "all" → wait for all in-sync replicas to acknowledge
        // "1"   → wait only for leader acknowledgement (faster, less durable)
        // "0"   → fire and forget (fastest, can lose messages)
        
        // Throughput optimisation
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        // Wait 5ms to batch messages before sending
        // Increases throughput at cost of 5ms latency
        
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        // Batch up to 16KB before sending
        
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        // Compress batches — reduces bandwidth, increases CPU usage
        
        // Reliability
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        // Exactly once semantics — no duplicate messages on retry
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**Kafka consumer configuration:**
```java
@KafkaListener(
    topics = "order-events",
    groupId = "notification-service",
    concurrency = "3"              // 3 threads, one per partition
)
public void handleOrderEvent(
        OrderEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
        @Header(KafkaHeaders.OFFSET) long offset) {
    
    log.info("Processing event at partition={} offset={}", partition, offset);
    
    try {
        notificationService.process(event);
        // Offset committed automatically after successful processing
    } catch (Exception e) {
        // Kafka has no DLQ natively — implement manually or use Spring's error handler
        errorHandler.handle(event, e);
    }
}
```

---

## RabbitMQ vs Kafka — The Decision Framework

| Dimension | RabbitMQ | Kafka |
|---|---|---|
| Core model | Message broker — route and delete | Distributed log — append and retain |
| Message retention | Deleted after consumption | Retained by time/size policy |
| Ordering | Per-queue ordering | Per-partition ordering |
| Throughput | High (tens of thousands/sec) | Very high (millions/sec) |
| Replay | Not supported natively | Core feature — replay from any offset |
| Consumer model | Push (broker pushes to consumer) | Pull (consumer polls broker) |
| Routing | Sophisticated (exchanges, bindings) | Simple (topic + partition key) |
| Complexity | Lower — simpler operational model | Higher — partitions, consumer groups, offsets |
| Best for | Task queues, RPC, complex routing | Event streaming, audit log, replay, high volume |

**Choose RabbitMQ when:**
- You need complex routing logic (different messages to different queues based on content)
- Task distribution — N workers sharing a queue of jobs
- Request/reply async pattern — built-in correlation and reply-to
- Simpler operational requirements
- Messages should be deleted once processed

**Choose Kafka when:**
- You need multiple independent consumers reading the same events
- Replay is required — rebuilding state, onboarding new services
- Very high throughput (millions of events/second)
- Event sourcing or audit log patterns
- Stream processing with Kafka Streams or Flink

---

## Message Delivery Guarantees — Critical for Interviews

```
At most once:
  Message sent once, consumer processes it once or not at all.
  How: auto-ack before processing
  Risk: consumer crashes after receiving but before processing → message lost
  Use: metrics, telemetry — losing occasional datapoints is acceptable

At least once:
  Message delivered until consumer acknowledges it.
  How: manual ack after processing
  Risk: consumer processes successfully but crashes before ACKing
       → redelivered → processed twice
  Requirement: consumers must be IDEMPOTENT
  Use: most business operations — prefer duplicates over loss

Exactly once:
  Message processed exactly one time, no duplicates, no loss.
  How: Kafka idempotent producer + transactional API
       or deduplication at consumer with idempotency keys
  Cost: higher latency, more complex
  Use: financial transactions, inventory updates
```

**Making consumers idempotent — the practical solution:**
```java
@KafkaListener(topics = "order-events")
public void handle(OrderConfirmedEvent event) {

    // Idempotency check — have we already processed this event?
    if (processedEventRepository.existsById(event.getMessageId())) {
        log.info("Duplicate event {}, skipping", event.getMessageId());
        return;  // already processed — safe to skip
    }
    
    // Process the event
    loyaltyService.awardPoints(event.getUserId(), event.getTotal());
    
    // Record that we processed it
    processedEventRepository.save(new ProcessedEvent(event.getMessageId()));
    
    // Now safe to ACK — even if redelivered, we will skip it
}
```

---

## The Outbox Pattern — Guaranteed Event Publishing

This is a Stage 7 pattern but essential to understand alongside message-based communication because it solves a critical problem:

**The problem:**
```java
// DANGEROUS — two operations, no atomicity
orderRepository.save(order);        // ① DB write succeeds
kafkaTemplate.send("order-events"); // ② Kafka publish fails

// Order saved but event never published
// Inventory never decremented, email never sent, loyalty never awarded
// System is in inconsistent state
```

**The Outbox Pattern — solve it with atomicity:**
```java
// SAFE — single DB transaction
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);           // ① save order
    outboxRepository.save(OutboxEvent      // ② save event to same DB
        .builder()
        .aggregateId(order.getId())
        .eventType("ORDER_CONFIRMED")
        .payload(serialize(order))
        .build());
    // Both succeed or both fail — atomic
}

// Separate outbox processor publishes to Kafka
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findUnpublished();
    for (OutboxEvent event : pending) {
        kafkaTemplate.send("order-events", event.getPayload());
        outboxRepository.markPublished(event.getId());
    }
}
```

The order and the event are saved atomically in the same DB transaction. The outbox processor reliably publishes them. If publishing fails, it retries. No inconsistency possible.

---

## Real-World Example — ShopSphere Message Architecture

```
ShopSphere message topology:

RabbitMQ (task queues):
  notification-queue        ← email/SMS jobs, processed by Notification Service workers
  payment-retry-queue       ← failed payment retries with TTL-based delay
  report-generation-queue   ← async report generation jobs

Kafka (event streaming):
  order-events              ← ORDER_CREATED, ORDER_CONFIRMED, ORDER_CANCELLED
    Consumers:
      notification-service-group  → confirmation email
      inventory-service-group     → stock decrement
      analytics-service-group     → revenue recording
      loyalty-service-group       → points award
      search-service-group        → order history index

  product-events            ← PRODUCT_CREATED, PRICE_UPDATED, STOCK_CHANGED
    Consumers:
      search-service-group        → Elasticsearch index update
      cdn-invalidation-group      → invalidate cached product pages
      recommendation-group        → retrain recommendation model

  payment-events            ← PAYMENT_SUCCESS, PAYMENT_FAILED, REFUND_PROCESSED
    Consumers:
      order-service-group         → update order status
      notification-service-group  → payment receipt or failure email
      analytics-service-group     → financial recording
```

**Rule of thumb for ShopSphere:**
- **Kafka** for domain events — things that happened that multiple services care about
- **RabbitMQ** for task queues — jobs that need exactly one worker to process them

---

## Interview Q&A

**Q: What is the difference between a queue and a topic in message-based communication?**
A queue delivers each message to exactly one consumer — multiple consumers compete to process messages, distributing the workload. A topic delivers each message to all subscriber groups — each group gets its own copy and processes it independently. Queues are for task distribution where work should happen once. Topics are for event broadcasting where multiple services need to react to the same event.

**Q: What is the difference between RabbitMQ and Kafka?**
RabbitMQ is a message broker — it routes messages to queues and deletes them after consumption. It excels at complex routing, task queues, and request/reply patterns. Kafka is a distributed append-only log — messages are retained after consumption and consumers track their own offsets. Kafka excels at high throughput event streaming, multiple independent consumers reading the same events, and replay — rebuilding state by re-reading historical events. Use RabbitMQ for work queues and routing logic, Kafka for event streams and audit logs.

**Q: What does at-least-once delivery mean and how do you handle it?**
At-least-once means the broker guarantees a message is delivered but may deliver it more than once — if a consumer processes the message but crashes before acknowledging, the broker redelivers it. The consumer must be idempotent — processing the same message twice produces the same result as processing it once. Implement this by checking a processed-event store before processing and recording completion after — duplicate deliveries are detected and skipped safely.

**Q: What is the Outbox Pattern and why is it needed?**
Without the Outbox Pattern, saving to the database and publishing an event are two separate operations with no atomicity — one can succeed while the other fails, leaving the system inconsistent. The Outbox Pattern saves the event to an outbox table in the same database transaction as the business entity — both succeed or both fail atomically. A separate processor then publishes events from the outbox to the message broker. This guarantees that every committed business operation eventually produces its corresponding event with no possibility of silent loss.

**Q: How does Kafka guarantee ordering?**
Kafka guarantees ordering within a partition — all messages in a single partition are delivered in the order they were written. Ordering across partitions is not guaranteed. To ensure all events for a given entity arrive in order, use that entity's ID as the partition key — all events for the same order, user, or product will always go to the same partition and be processed in sequence. The number of partitions determines the maximum parallelism — each partition can be consumed by at most one consumer within a group.

---

Say **"next"** when ready for Topic 7 — Publisher-Subscriber Model.
