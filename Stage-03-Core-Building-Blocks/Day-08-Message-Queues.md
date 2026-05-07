# Stage 3 — Topic 6: Message Queues (Deep Dive)

## Theory

We covered message-based communication conceptually in Stage 2 Topic 6. Now we go deep on the internals — Kafka's log architecture, partition mechanics, consumer group rebalancing, exactly-once semantics, RabbitMQ's exchange internals, and the operational realities of running message queues in production.

This is the level of depth that distinguishes a senior engineer answer from a mid-level one. Knowing "Kafka is a distributed log" is table stakes. Understanding **why** it makes the choices it does — and what those choices cost — is what gets you hired.

**The fundamental question message queues answer:**
```
How do you connect two services that produce and consume
data at different rates, with different availability,
and without either one knowing about the other?

Producer: 10,000 orders/second during flash sale
Consumer: can process 2,000 orders/second

Without a queue:
  Producer overwhelms consumer → consumer crashes
  Consumer is down → producer loses data
  
With a queue:
  Producer writes at 10,000/second → queue buffers
  Consumer reads at 2,000/second → works through backlog
  Consumer restarts → resumes from where it left off
  Neither side needs to know the other exists
```

---

## Kafka — Deep Internals

### The Log — Kafka's Core Abstraction

Everything in Kafka is built on one idea: **an append-only, immutable, ordered sequence of records**.

```
Topic: order-events
Partition 0:

Offset: 0      1      2      3      4      5      6      7
        │      │      │      │      │      │      │      │
        ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
      [e0]   [e1]   [e2]   [e3]   [e4]   [e5]   [e6]   [e7]
       ↑ oldest                                    ↑ newest
       
Rules:
  Records are APPENDED only — never modified or deleted in place
  Each record has a monotonically increasing OFFSET
  Offset is assigned by the partition — globally unique within partition
  Records are retained for a configurable period (time or size-based)
  Reading does NOT delete records — consumers track their own offset
```

**Why immutability matters:**
```
Multiple consumers read independently:
  Consumer Group A (Notification): at offset 5
  Consumer Group B (Analytics):    at offset 2 (behind — processing slowly)
  Consumer Group C (New service):  at offset 0 (reading history)

None affects the others.
Analytics being slow does not block Notification.
New service can read from the beginning of history.

Compare to a traditional queue:
  Message consumed → deleted
  Cannot have multiple independent consumers
  Cannot replay history
  Cannot add a new consumer and process historical data
```

### Partition Internals — How Data Is Stored on Disk

```
Each partition is a directory on disk:
  /kafka-logs/order-events-0/
    00000000000000000000.log      ← segment file (actual messages)
    00000000000000000000.index    ← offset index (offset → file position)
    00000000000000000000.timeindex← timestamp index (timestamp → offset)
    00000000000000001000.log      ← next segment (created after rollover)
    00000000000000001000.index
    leader-epoch-checkpoint

Segment files:
  Partition log is split into segment files
  Active segment: being written to
  Closed segments: read-only, eligible for deletion when retention exceeded
  
  Default segment size: 1GB or 7 days (whichever comes first)
  
Index file structure:
  Sparse index — not every offset has an entry
  Maps offset → byte position in log file
  
  Lookup: offset 5000 → binary search in index → byte position 204800
          → seek directly to byte 204800 in log file
          → read record
          
  Without index: linear scan through entire log file — O(n)
  With index:    binary search + single seek — O(log n)
```

### Producer Internals — How Records Are Published

```java
// Kafka Producer lifecycle
KafkaProducer<String, OrderEvent> producer = new KafkaProducer<>(config);

ProducerRecord<String, OrderEvent> record = new ProducerRecord<>(
    "order-events",      // topic
    "o-789",             // key (determines partition)
    orderConfirmedEvent  // value
);

Future<RecordMetadata> future = producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // Handle send failure
        log.error("Failed to send record", exception);
    } else {
        log.info("Sent to partition={} offset={}",
            metadata.partition(), metadata.offset());
    }
});
```

**Producer batching pipeline:**
```
producer.send(record) called:
  ↓
Record added to in-memory RecordAccumulator (per-partition buffer)
  ↓
Sender thread runs in background:
  When batch full (batch.size bytes) → send immediately
  OR when linger.ms elapsed         → send whatever is batched

  linger.ms=0:   send immediately — lowest latency, smallest batches
  linger.ms=5:   wait 5ms to fill batch — better throughput
  linger.ms=100: wait 100ms — highest throughput, higher latency

  After batch sent to broker:
    Broker writes to partition log
    If acks=all: waits for all in-sync replicas to write
    Returns RecordMetadata (partition, offset, timestamp)
```

**Producer acknowledgement levels — the durability trade-off:**
```
acks=0  (fire and forget):
  Producer sends, does not wait for any acknowledgement
  Highest throughput — no waiting
  Data loss possible if broker crashes before writing
  Use: metrics, telemetry where occasional loss is acceptable

acks=1  (leader ack):
  Producer waits for leader broker to write to its log
  Moderate throughput
  Data loss possible if leader crashes before replicating to followers
  Use: non-critical events where throughput matters more than durability

acks=all (acks=-1):
  Producer waits for ALL in-sync replicas (ISR) to acknowledge
  Lowest throughput — must wait for slowest replica
  No data loss as long as at least one ISR remains
  Use: financial transactions, orders, any critical business events
  
ShopSphere: acks=all for order-events, payment-events
            acks=1  for analytics-events, log-events
```

**Idempotent producer — exactly-once at the producer level:**
```java
// Enable idempotent producer
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// Also sets: acks=all, retries=MAX_INT, max.in.flight.requests=5

// How it works:
// Producer gets a unique ProducerID (PID) from broker
// Each record gets a sequence number per partition
// Broker deduplicates records with same PID + sequence number
// Retries cannot cause duplicates — broker tracks sequence

// Without idempotence:
//   Producer sends record → network timeout → producer retries
//   → Broker received original AND retry → duplicate in log

// With idempotence:
//   Producer sends record → network timeout → producer retries  
//   → Broker checks: PID=42, seq=100 already received → skip duplicate
//   → Exactly once delivery at producer level
```

---

### Consumer Internals — Offset Management

**How consumers track position:**
```
Consumer does not receive a message — it POLLS for messages:

while (true) {
    ConsumerRecords<String, OrderEvent> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, OrderEvent> record : records) {
        processRecord(record);
        consumer.commitSync();  // commit offset after processing
    }
}

Poll returns a batch of records from assigned partitions.
Consumer tracks offset per partition: next record to read.
Offset committed to __consumer_offsets internal topic.
```

**Auto commit vs manual commit:**
```
Auto commit (enable.auto.commit=true):
  Kafka commits offsets every auto.commit.interval.ms (default 5000ms)
  Simple but dangerous:
    Records polled at t=0
    Auto commit fires at t=5000ms
    Consumer crashes at t=2500ms — records processed but offset not committed
    On restart: consumer re-reads from last committed offset
    Records processed TWICE — at-least-once semantics
    
    OR worse:
    Auto commit fires at t=5000ms — offsets committed
    Consumer crashes at t=6000ms — some records in batch not processed
    On restart: consumer skips unprocessed records
    Records lost — at-most-once semantics

Manual commit (enable.auto.commit=false):
  Application controls exactly when offsets are committed

commitSync():    Blocks until broker confirms commit
                 Slower but reliable
                 
commitAsync():   Non-blocking — fires and continues
                 Faster but no guarantee commit succeeded
                 Use completion callback to handle failures
```

**Commit strategies — at-least-once with idempotent consumer:**
```java
@KafkaListener(topics = "order-events", groupId = "inventory-service")
public void handleOrderEvent(
        ConsumerRecord<String, OrderEvent> record,
        Acknowledgment ack) {  // manual ack mode

    OrderEvent event = record.value();
    String messageId = event.getEventId();

    // 1. Idempotency check
    if (processedEventRepository.existsById(messageId)) {
        log.info("Duplicate event {}, skipping", messageId);
        ack.acknowledge();  // commit offset — skip cleanly
        return;
    }

    try {
        // 2. Process event
        inventoryService.processOrderEvent(event);

        // 3. Mark as processed
        processedEventRepository.save(new ProcessedEvent(messageId));

        // 4. Commit offset — only AFTER successful processing
        ack.acknowledge();

    } catch (RetryableException e) {
        // Don't acknowledge — Kafka will redeliver
        log.warn("Retryable error for event {}, will retry", messageId);
        throw e;

    } catch (NonRetryableException e) {
        // Send to DLQ, then acknowledge to not block partition
        deadLetterService.send(record, e);
        ack.acknowledge();
    }
}
```

---

### Consumer Groups — Partitions and Parallelism

**The fundamental rule:**
```
A partition can be assigned to AT MOST ONE consumer within a group.
A consumer can be assigned MULTIPLE partitions.

Topic: order-events with 6 partitions

Consumer Group: inventory-service
  Consumer 1: Partition 0, Partition 1
  Consumer 2: Partition 2, Partition 3
  Consumer 3: Partition 4, Partition 5
  → Perfect distribution, full parallelism

Add Consumer 4:
  Consumer 1: Partition 0
  Consumer 2: Partition 1, Partition 2
  Consumer 3: Partition 3, Partition 4
  Consumer 4: Partition 5
  → Rebalanced — all consumers active

Add Consumer 7 (more consumers than partitions):
  Consumer 1–6: one partition each
  Consumer 7: NO PARTITION — idle
  → Cannot exceed parallelism beyond partition count

  To increase parallelism: increase partition count
  (Cannot be reduced — increasing partitions is irreversible)
```

**Consumer group rebalancing — what happens:**
```
Trigger events:
  Consumer joins the group
  Consumer leaves (crash or graceful shutdown)
  Partition count changes
  Consumer heartbeat times out

Rebalance process (Stop-the-world):
  1. Group Coordinator (broker) detects trigger
  2. Sends JoinGroup request to all consumers
  3. All consumers STOP consuming — pause
  4. One consumer elected as Group Leader
  5. Leader receives member list + partition list
  6. Leader runs partition assignment algorithm
  7. Leader sends assignments to Coordinator
  8. Coordinator distributes assignments to all consumers
  9. Consumers resume consuming from new partitions

During rebalance: zero consumption — processing stops
Typical duration: 2-10 seconds for small groups
Problem at scale: frequent consumer churn = frequent rebalances

Solutions:
  Static group membership: consumer gets stable ID
    → Rejoining consumer gets same partitions back
    → No rebalance on restart (within session.timeout.ms)
    
  Incremental Cooperative Rebalancing (Kafka 2.4+):
    Only reassign the partitions that NEED to move
    Other consumers continue processing unaffected partitions
    Dramatically reduces pause duration
```

---

### Kafka Replication — How Durability Works

```
Topic partition with replication factor 3:

Brokers: broker-1, broker-2, broker-3

Partition 0:
  Leader:   broker-1  ← producers write here, consumers read here
  Follower: broker-2  ← replicates from leader
  Follower: broker-3  ← replicates from leader

In-Sync Replicas (ISR):
  Set of replicas fully caught up with leader
  Initially: ISR = {broker-1, broker-2, broker-3}
  
  If broker-2 falls behind (slow disk, GC pause):
    Removed from ISR after replica.lag.time.max.ms
    ISR = {broker-1, broker-3}
  
  With acks=all:
    Leader writes record
    Waits for ALL ISR replicas to confirm write
    Returns ack to producer only when all ISR confirms
    
  If leader broker-1 crashes:
    Controller (ZooKeeper/KRaft) detects leader failure
    Elects new leader from ISR — broker-2 or broker-3
    Clients redirect to new leader
    Recovery time: ~30 seconds (controlled failover)
    
  min.insync.replicas=2:
    Production config — refuse writes if ISR < 2
    Prevents data loss if leader crashes after ack
    Producer gets NotEnoughReplicasException — must retry
```

---

## RabbitMQ — Deep Internals

### Exchange Types in Depth

```
Message flow:
  Producer → Exchange → [Binding Rules] → Queue → Consumer

Exchange does NOT store messages.
Queue stores messages.
Binding is the routing rule between exchange and queue.
```

**Direct Exchange — exact routing key match:**
```
Exchange: order-events (type: direct)
Bindings:
  routing_key="order.confirmed" → queue: notification-queue
  routing_key="order.cancelled" → queue: refund-queue
  routing_key="order.confirmed" → queue: inventory-queue (multiple bindings OK)

Producer sends:
  message with routing_key="order.confirmed"
  → Delivered to: notification-queue AND inventory-queue

  message with routing_key="order.cancelled"
  → Delivered to: refund-queue only
  
  message with routing_key="order.shipped"
  → No binding → DROPPED (or returned to producer if mandatory=true)
```

**Topic Exchange — pattern matching:**
```
Exchange: shopsphere-events (type: topic)
Routing key format: <domain>.<entity>.<action>

Bindings:
  "order.*"         → order-all-queue       (all order events)
  "order.confirmed" → notification-queue    (exact match)
  "*.failed"        → alerts-queue          (any failed event)
  "#"               → audit-queue           (everything)
  "payment.#"       → payment-queue         (all payment events)

Words:  * matches exactly one word
#:      matches zero or more words

Producer sends routing_key="payment.charge.failed":
  → audit-queue        (# matches everything)
  → alerts-queue       (*.failed? No — *.failed matches one.failed, not payment.charge.failed)
  → payment-queue      (payment.# matches payment.charge.failed ✓)
```

**Fanout Exchange — broadcast:**
```
Exchange: broadcast-events (type: fanout)
Bindings: ALL bound queues — routing key IGNORED

Every message goes to every bound queue.
Use: system-wide notifications, cache invalidation,
     broadcasting to all microservice instances.
```

**Headers Exchange — attribute-based routing:**
```
Exchange: routed-events (type: headers)
Bindings use header attributes, not routing key:

Binding 1:
  x-match: all       (all headers must match)
  region: india
  priority: high
  → india-high-priority-queue

Binding 2:
  x-match: any       (any header match is sufficient)
  region: india
  region: uk
  → india-uk-queue

Message headers:
  region: india, priority: high → india-high-priority-queue ✓
  region: india, priority: low  → india-uk-queue (any match: region=india) ✓
```

### Queue Properties — Durability and Behaviour

```java
// Declaring a durable queue in Spring Boot
@Configuration
public class RabbitConfig {

    @Bean
    public Queue orderNotificationQueue() {
        return QueueBuilder
            .durable("order.notifications")    // survives broker restart
            .withArgument("x-dead-letter-exchange", "dlx.order")
            .withArgument("x-dead-letter-routing-key", "order.notifications.dlq")
            .withArgument("x-message-ttl", 86400000)  // messages expire after 24h
            .withArgument("x-max-length", 100000)     // max 100K messages
            .withArgument("x-overflow", "reject-publish")  // reject when full
            .build();
    }

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder
            .topicExchange("order-events")
            .durable(true)
            .build();
    }

    @Bean
    public Binding notificationBinding() {
        return BindingBuilder
            .bind(orderNotificationQueue())
            .to(orderExchange())
            .with("order.confirmed");
    }

    // Dead Letter Queue — receives failed/expired messages
    @Bean
    public Queue orderNotificationDLQ() {
        return QueueBuilder
            .durable("order.notifications.dlq")
            .build();
    }
}
```

**Queue durability modes:**
```
Durable queue:
  Queue definition survives broker restart
  Messages survive broker restart ONLY if also marked persistent
  
Transient queue:
  Deleted when broker restarts
  Use for: temporary RPC reply queues, ephemeral work items
  
Message persistence:
  deliveryMode=2 (persistent): message written to disk before ack
  deliveryMode=1 (transient):  message in memory only
  
  Persistent message + durable queue = survives broker crash
  Cost: disk write per message — lower throughput
  
Exclusive queue:
  Bound to one connection
  Deleted when connection closes
  Use for: private reply-to queues in request-reply pattern
```

### Dead Letter Queue — Production Pattern

```
Message becomes a dead letter when:
  1. Consumer rejects it with requeue=false
  2. Message TTL expires before consumption
  3. Queue length limit exceeded (x-max-length)

Dead Letter Exchange (DLX) routes dead letters to DLQ:

Normal flow:
  Producer → order-events exchange → order.notifications queue → Consumer

Failure flow:
  Consumer rejects 3 times → message dead-lettered
  → Dead Letter Exchange (dlx.order)
  → order.notifications.dlq queue
  → Ops team investigates, fixes, republishes

```java
@RabbitListener(queues = "order.notifications")
public void handleOrderNotification(
        OrderConfirmedEvent event,
        Channel channel,
        @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {

    try {
        notificationService.sendConfirmation(event);
        channel.basicAck(tag, false);  // success

    } catch (TemporaryException e) {
        // Requeue for retry
        channel.basicNack(tag, false, true);

    } catch (PermanentException e) {
        // Don't requeue — send to DLQ
        channel.basicNack(tag, false, false);
        // Message goes to DLX → DLQ automatically
    }
}

// DLQ consumer — ops team monitoring
@RabbitListener(queues = "order.notifications.dlq")
public void handleDeadLetter(
        Message message,
        @Header("x-death") List<Map<String, Object>> deathHeaders) {

    String originalQueue = (String) deathHeaders.get(0).get("queue");
    String reason = (String) deathHeaders.get(0).get("reason");
    long retryCount = (long) deathHeaders.get(0).get("count");

    log.error("Dead letter from queue={} reason={} retries={}",
        originalQueue, reason, retryCount);

    alertService.notifyOps(message, reason);
}
```

---

## Kafka vs RabbitMQ — When to Use Each

```
Kafka is better when:

1. Multiple independent consumers need the same events:
   order.confirmed → notification + inventory + analytics + loyalty
   All four read independently, at their own pace.
   RabbitMQ: need 4 separate queues fed by fanout exchange
   Kafka: one topic, four consumer groups — built for this

2. Replay is needed:
   New analytics service needs 6 months of order history.
   Kafka: seek to oldest offset → replay all 6 months
   RabbitMQ: messages deleted after consumption → impossible

3. Very high throughput:
   Kafka: millions of messages/second
   RabbitMQ: hundreds of thousands/second

4. Event sourcing / audit log:
   Log is the storage format — Kafka is naturally aligned

5. Stream processing:
   Kafka Streams, Flink reading from Kafka natively


RabbitMQ is better when:

1. Complex routing logic:
   Route by content, headers, patterns → RabbitMQ exchanges excel
   Kafka: all routing is by partition key — simple

2. Task queues with competing consumers:
   One message, one worker — classic work queue pattern
   Multiple workers share queue load automatically
   
3. Request-reply async pattern:
   Built-in correlation ID + reply-to queue support
   
4. Per-message TTL and priority:
   Priority queues (x-max-priority)
   Per-message TTL
   Kafka: no native priority, TTL is partition-level only

5. Simpler operational model:
   Smaller teams, less infrastructure expertise
   Easier to reason about for simple use cases

6. When messages should truly be deleted after processing:
   Privacy requirements — GDPR deletion
   No need to store historical events
```

---

## Ordering Guarantees — A Critical Detail

```
Kafka:
  Ordering guaranteed WITHIN a partition.
  NOT guaranteed across partitions.
  
  To guarantee order for a logical entity:
    Use entity ID as partition key
    hash("order:o-789") → always goes to Partition 2
    All events for order o-789 are in order on Partition 2
    
  If multiple partitions receive events for same entity:
    ORDER_CREATED  → Partition 0 (processed first)
    ORDER_CONFIRMED→ Partition 3 (processed after!)
    Wrong — consumer sees CONFIRMED before CREATED
    
RabbitMQ:
  Ordering guaranteed within a queue with ONE consumer.
  Multiple consumers on same queue → no order guarantee.
  
  If order matters: one consumer per queue, or
                    partition messages to separate queues by entity

Real implication for ShopSphere:
  Order lifecycle events must use orderId as Kafka partition key
  All events for same order → same partition → guaranteed ordering
  Inventory Service correctly processes: CREATED → CONFIRMED → CANCELLED
  Never processes: CANCELLED before CONFIRMED (impossible in same partition)
```

---

## Exactly-Once Semantics — The Hard Problem

```
At-most-once:   Consumer may miss messages (committed before processing)
At-least-once:  Consumer may process duplicates (committed after processing)
Exactly-once:   Process each message exactly once — hardest to achieve

Kafka exactly-once (end-to-end):

1. Idempotent producer:
   Producer retries cannot create duplicates
   (PID + sequence number deduplication at broker)

2. Transactional producer:
   Multiple writes to multiple partitions atomically
   All-or-nothing — either all writes committed or none

3. Read-process-write atomically:
   Consumer reads from topic A
   Processes record
   Writes to topic B
   Commits offset to topic A
   All three atomic — cannot have partial success

```java
// Transactional producer — exactly-once
producer.initTransactions();

try {
    producer.beginTransaction();

    // Write to multiple partitions atomically
    producer.send(new ProducerRecord<>("order-events",
        "o-789", orderConfirmedEvent));
    producer.send(new ProducerRecord<>("notification-events",
        "u-123", notifyUserEvent));

    // Commit offset atomically with the produce
    producer.sendOffsetsToTransaction(
        Collections.singletonMap(
            new TopicPartition("input-topic", 0),
            new OffsetAndMetadata(record.offset() + 1)
        ),
        consumer.groupMetadata()
    );

    producer.commitTransaction();

} catch (ProducerFencedException e) {
    producer.close();
} catch (Exception e) {
    producer.abortTransaction();  // rollback all writes
}
```

**Practical exactly-once with idempotent consumers:**
```
In practice, true Kafka exactly-once is complex and has overhead.
Most production systems use:
  At-least-once delivery (simple, reliable)
  + Idempotent consumers (deduplicate at application level)

This achieves the same effect with less complexity:
  If message delivered twice:
    Consumer checks: already processed this eventId? → skip
  Result: business operation happens exactly once

This is the correct answer for 99% of interview questions about
exactly-once in Kafka — mention both approaches.
```

---

## Real-World Example — ShopSphere Message Architecture Deep Dive

```java
// Complete Kafka configuration for ShopSphere

@Configuration
public class KafkaConfig {

    // Producer — critical events
    @Bean("criticalProducer")
    public KafkaTemplate<String, Object> criticalKafkaTemplate() {
        Map<String, Object> config = Map.of(
            BOOTSTRAP_SERVERS_CONFIG,   "kafka-1:9092,kafka-2:9092,kafka-3:9092",
            KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class,
            VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class,
            ACKS_CONFIG,                "all",           // wait for all ISR
            RETRIES_CONFIG,             3,
            ENABLE_IDEMPOTENCE_CONFIG,  true,            // exactly-once at producer
            LINGER_MS_CONFIG,           0,               // no batching delay
            COMPRESSION_TYPE_CONFIG,    "snappy"
        );
        return new KafkaTemplate<>(new DefaultKafkaProducerFactory<>(config));
    }

    // Producer — analytics (throughput optimised)
    @Bean("analyticsProducer")
    public KafkaTemplate<String, Object> analyticsKafkaTemplate() {
        Map<String, Object> config = Map.of(
            BOOTSTRAP_SERVERS_CONFIG,    "kafka-1:9092,kafka-2:9092,kafka-3:9092",
            KEY_SERIALIZER_CLASS_CONFIG,  StringSerializer.class,
            VALUE_SERIALIZER_CLASS_CONFIG,JsonSerializer.class,
            ACKS_CONFIG,                 "1",             // leader ack only
            LINGER_MS_CONFIG,            5,               // batch for 5ms
            BATCH_SIZE_CONFIG,           32768,           // 32KB batch
            COMPRESSION_TYPE_CONFIG,     "snappy"
        );
        return new KafkaTemplate<>(new DefaultKafkaProducerFactory<>(config));
    }

    // Consumer — notification service
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            notificationListenerFactory() {

        Map<String, Object> config = Map.of(
            BOOTSTRAP_SERVERS_CONFIG,       "kafka-1:9092,kafka-2:9092,kafka-3:9092",
            GROUP_ID_CONFIG,                "notification-service",
            KEY_DESERIALIZER_CLASS_CONFIG,   StringDeserializer.class,
            VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class,
            AUTO_OFFSET_RESET_CONFIG,        "earliest",  // read from beginning if new group
            ENABLE_AUTO_COMMIT_CONFIG,       false,       // manual commit
            MAX_POLL_RECORDS_CONFIG,         50,          // process 50 records per poll
            SESSION_TIMEOUT_MS_CONFIG,       30000,
            HEARTBEAT_INTERVAL_MS_CONFIG,    3000
        );

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(new DefaultKafkaConsumerFactory<>(config));
        factory.setConcurrency(3);  // 3 threads = consume 3 partitions in parallel
        factory.getContainerProperties()
            .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Error handling — send to DLQ after 3 retries
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(criticalKafkaTemplate()),
            new FixedBackOff(1000L, 3L)  // retry 3 times, 1 second apart
        ));

        return factory;
    }
}

// Topic configuration — created programmatically
@Bean
public NewTopic orderEventsTopic() {
    return TopicBuilder.name("order-events")
        .partitions(12)          // 12 partitions = 12 max parallel consumers
        .replicas(3)             // replicate to 3 brokers
        .config(TopicConfig.RETENTION_MS_CONFIG,
            String.valueOf(Duration.ofDays(7).toMillis()))
        .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
        .build();
}
```

---

## Interview Q&A

**Q: What is Kafka's offset and why is it central to how Kafka works?**
An offset is a monotonically increasing integer assigned to each record within a partition — it uniquely identifies a record's position in the partition log. Consumers track their progress by storing the offset of the last record they processed. Because reading does not delete records, multiple independent consumer groups can read the same partition at different offsets simultaneously without affecting each other. If a consumer crashes it resumes from its committed offset — no data is lost. This is the fundamental difference from traditional message queues where consumption deletes the message.

**Q: How many Kafka consumers can you have in a consumer group?**
The maximum effective parallelism in a consumer group equals the number of partitions in the topic — each partition is assigned to at most one consumer in the group. If you have 6 partitions and 6 consumers, each consumer processes one partition. If you add a 7th consumer it sits idle with no partition assigned. To increase parallelism you must increase partition count. This is why choosing partition count upfront matters — you can increase partitions later but cannot decrease them, and increasing causes a rebalance.

**Q: What is a Kafka consumer group rebalance and why is it problematic at scale?**
A rebalance is triggered when a consumer joins or leaves a group, causing Kafka to reassign partitions among all active consumers. During a rebalance all consumers in the group stop processing — a stop-the-world pause that can last seconds. At scale with many consumers, frequent crashes or rolling deployments cause constant rebalances, creating processing gaps. Mitigations are static group membership where consumers are assigned stable IDs so rejoining gets the same partitions without triggering a full rebalance, and incremental cooperative rebalancing which only moves partitions that need to change rather than reassigning everything.

**Q: What is the difference between acks=1 and acks=all in Kafka?**
With acks=1 the producer waits only for the partition leader to write the record and acknowledge — faster but if the leader crashes before replicating to followers, the record is lost. With acks=all the producer waits for all in-sync replicas to acknowledge the write — no record is lost as long as at least one in-sync replica survives a leader failure. The cost is latency — the producer must wait for the slowest in-sync replica. Production systems use acks=all for critical events like orders and payments, and acks=1 for non-critical high-throughput streams like analytics where occasional loss is acceptable.

**Q: How do you achieve exactly-once processing in Kafka?**
True Kafka exactly-once requires three components working together — idempotent producer which uses sequence numbers so retries cannot create duplicates, transactional API which allows atomic writes across multiple partitions and atomic offset commits, and idempotent consumers which check a processed-event store before processing and record completion after. In practice most production systems use at-least-once delivery combined with idempotent consumers — simpler, lower overhead, and achieves the same business outcome because duplicate deliveries are detected and skipped at the application level.

---

## Stage 3 Progress

```
✅ Topic 1 — Load Balancers
✅ Topic 2 — Forward Proxy vs Reverse Proxy
✅ Topic 3 — CDN
✅ Topic 4 — Caching
✅ Topic 5 — Consistent Hashing
✅ Topic 6 — Message Queues (Deep Dive)
⬜ Topic 7 — SQL vs NoSQL
⬜ Topic 8 — Object Storage
⬜ Topic 9 — Different Storage Systems
```

---

Say **"next"** when ready for Topic 7 — SQL vs NoSQL.
