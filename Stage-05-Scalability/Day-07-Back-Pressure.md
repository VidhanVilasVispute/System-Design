# Topic 7 — Back-pressure

## The Core Problem

```
Every system has a speed mismatch somewhere.

Producer generates work faster than consumer can process it.

Without any signal back:
  Producer: "No errors? Keep sending!" → sends faster
  Consumer: queue fills → memory exhausts → crashes
  Producer: "Consumer crashed? Retry!" → makes it worse

Back-pressure = the mechanism by which a downstream component
                signals "I am overwhelmed — slow down"
                to its upstream.

Without back-pressure: fast producer kills slow consumer.
With back-pressure:    system self-regulates to sustainable throughput.
```

---

## The Water Pipe Analogy

```
NO BACK-PRESSURE:

  High pressure ──────────────────────────────▶ Small pipe
  water source   ████████████████████████████   (consumer)
                 ↑ keeps pushing regardless        │
                                                   ▼
                                              PIPE BURSTS 💥

WITH BACK-PRESSURE:

  Water source ◀── pressure signal ── Small pipe
      │             "I'm full,          (consumer)
      │              slow down"
      ▼
  Reduces flow to match pipe capacity
  System reaches equilibrium ✅
```

---

## Where Back-pressure Lives in a Distributed System

```
Three places it manifests:

1. Thread/process level:
   Bounded queues inside your JVM
   When queue full → caller blocks or gets rejected

2. Service-to-service level:
   HTTP 429, 503 responses
   gRPC flow control
   Timeouts and circuit breakers

3. Message queue level:
   Consumer fetch rate controls producer indirectly
   Kafka consumer lag as a signal
   RabbitMQ prefetch count

ShopSphere has all three. We go through each.
```

---

## Level 1 — Thread Level Back-pressure (Inside JVM)

### The Unbounded Queue Problem

```java
// DANGEROUS — no back-pressure:
ExecutorService executor = Executors.newFixedThreadPool(10);

// Order events arrive at 10,000/sec
// Thread pool processes at 100/sec
// Queue grows: 100 → 1000 → 100,000 → OutOfMemoryError 💥

while (true) {
    OrderEvent event = kafka.poll();
    executor.submit(() -> processOrder(event));  // queue grows unbounded
}
```

```
Memory:
  t=0s:    queue = 0 tasks    (10MB)
  t=10s:   queue = 99,000     (500MB)
  t=20s:   queue = 198,000    (1GB)
  t=30s:   OutOfMemoryError   💥 JVM dies
```

### The Fix — Bounded Queue + Rejection Policy

```java
// CORRECT — back-pressure via bounded queue:
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // core threads
    10,                              // max threads
    0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(1000),  // BOUNDED queue — max 1000 tasks
    new ThreadPoolExecutor.CallerRunsPolicy()  // back-pressure policy
);
```

```
CallerRunsPolicy:
  Queue full (1000 tasks waiting)?
  → The CALLING thread executes the task itself
  → Calling thread is busy → it can't submit more tasks
  → Producer naturally slows down to consumer speed
  → Back-pressure achieved ✅

Other rejection policies:
  AbortPolicy:         throw RejectedExecutionException → caller handles it
  DiscardPolicy:       silently drop task (data loss!)
  DiscardOldestPolicy: drop oldest queued task, add new one
  CallerRunsPolicy:    caller thread does the work → slows producer ✅
```

```
Visualized:

  Producer thread         Bounded Queue (cap=1000)    Worker threads (10)
  ──────────────          ──────────────────────      ────────────────────
  submit task 1    ──▶    [████████░░░░░░░░░░░░]  ──▶  thread-1: processing
  submit task 2    ──▶    [█████████░░░░░░░░░░░]  ──▶  thread-2: processing
  ...
  submit task 1000 ──▶    [████████████████████]  ──▶  thread-10: processing
  submit task 1001        QUEUE FULL
       │
       ▼ CallerRunsPolicy
  Producer thread RUNS task 1001 itself
  → Producer busy → can't call submit → natural throttle
```

---

## Level 2 — Service-to-Service Back-pressure (HTTP)

### HTTP Status Codes as Back-pressure Signals

```
503 Service Unavailable:
  "I am overloaded right now. Try later."
  Retry-After: 30   ← tells caller when to retry

429 Too Many Requests:
  "You specifically are sending too fast."
  Retry-After: 10

These are explicit back-pressure signals.
A well-behaved client reads them and backs off.
```

### The Problem: Clients That Ignore Back-pressure

```
order-service ──▶ payment-service

payment-service returns 503 (overwhelmed)

BAD client (no back-pressure handling):
  retry immediately → retry immediately → retry immediately
  → makes payment-service MORE overwhelmed
  → 503 → retry → 503 → retry → death spiral

GOOD client (back-pressure aware):
  503 received
  → wait Retry-After seconds
  → exponential backoff if no Retry-After header
  → circuit breaker opens after N failures
  → stops sending entirely for a cooldown period
```

### Circuit Breaker — The Systematic Back-pressure Pattern

```
Circuit Breaker States:

  CLOSED (normal):
    Requests flow through
    Failure counter ticks on each error
    
  ┌─────────────────────────────────────────────────────┐
  │  CLOSED        failure threshold exceeded           │
  │  (normal) ─────────────────────────────────────────▶│  OPEN
  │                                                     │  (blocking)
  │  ◀──────────────────────────────────────────────────│
  │  success rate OK         half-open: 1 test request  │
  │                          passes?                    │
  └─────────────────────────────────────────────────────┘

  CLOSED:
    All requests flow to downstream
    Monitor failure rate

  OPEN:
    Downstream is failing — stop sending IMMEDIATELY
    Return fallback response to caller
    No requests reach downstream → it gets breathing room to recover
    Wait cooldown period (e.g., 30s)

  HALF-OPEN:
    After cooldown, let ONE request through
    Succeeds? → CLOSED (downstream recovered)
    Fails?    → OPEN again (not ready yet)
```

```java
// Resilience4j Circuit Breaker in ShopSphere order-service:
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentServiceClient.process(request);
}

public PaymentResult paymentFallback(PaymentRequest request, Exception e) {
    // Back-pressure handled: payment-service is down
    // Don't hammer it — queue the payment for async retry
    pendingPaymentQueue.add(request);
    return PaymentResult.pending("Payment queued, will process shortly");
}
```

```yaml
# application.yml — circuit breaker config:
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowSize: 10          # last 10 requests
        failureRateThreshold: 50       # open if 50%+ fail
        waitDurationInOpenState: 30s   # cooldown period
        permittedCallsInHalfOpenState: 1
```

```
Why circuit breaker IS back-pressure:
  Downstream overwhelmed → failures spike → CB opens
  Upstream STOPS sending → downstream gets zero new load
  Downstream recovers (drains its queue)
  CB half-opens → tests → recovers → CB closes
  
  The CB is the upstream's response to the downstream's distress signal.
```

---

## Level 3 — Message Queue Back-pressure (Kafka & RabbitMQ)

This is the richest and most important form for microservices.

### Kafka — Consumer Lag as Back-pressure Signal

```
Kafka's design is inherently back-pressure friendly:

  Producer writes to partition log (broker)
  Consumer reads at ITS OWN PACE
  Broker doesn't push to consumer — consumer PULLS

  ┌──────────────────────────────────────────────────────┐
  │  Partition 0 log (broker)                           │
  │                                                      │
  │  offset: 0  1  2  3  4  5  6  7  8  9  10           │
  │             ●  ●  ●  ●  ●  ●  ●  ●  ●  ●            │
  │                                        ↑             │
  │                                   latest offset      │
  │                      ↑                               │
  │                 consumer offset                      │
  │                 (consumer is here)                   │
  │                                                      │
  │  Consumer lag = latest offset - consumer offset = 4  │
  └──────────────────────────────────────────────────────┘

Consumer is behind by 4 messages = lag = 4.

Consumer naturally reads only what it can handle:
  Process 10ms/msg → reads at 100msg/sec
  Producer writes 50msg/sec → lag grows at 40msg/sec

This growing lag IS the back-pressure signal.
```

```
What to do with consumer lag:

  Lag = 0:          consumer keeping up ✅
  Lag growing:      consumer falling behind — scale up consumers
  Lag = 1,000,000:  consumer crashed or too slow — alert!

Kafka auto-scaling via KEDA (in K8s):

  KEDA watches consumer group lag.
  Lag > 1000 → scale notification-service from 2 pods to 5 pods.
  Lag = 0    → scale back down to 2 pods.

  This IS automated back-pressure response:
    Kafka signals overwhelm via lag metric
    KEDA reacts by adding processing capacity
```

### Consumer Fetch Rate Controls

```java
// Kafka consumer — back-pressure via fetch configuration:
@KafkaListener(topics = "order-placed-events")
public void handleOrderPlaced(OrderPlacedEvent event) {
    // This method IS the back-pressure mechanism:
    // Kafka only fetches the next batch AFTER this method returns.
    // If processing is slow → fetch rate drops → producer/broker not overloaded.
    
    notificationService.sendOrderConfirmation(event);  // takes 50ms
    // Next message fetched only after this returns
}
```

```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 10          # fetch max 10 records per poll
      fetch-max-wait: 500ms         # wait up to 500ms for 10 records
      # If processing 10 records takes 2s → effective rate = 5 records/sec
      # Kafka broker accumulates the rest → consumer lag → back-pressure signal
```

```
max-poll-records is critical:
  Too high (e.g., 500):
    Consumer fetches 500 records, takes 50s to process them
    Heartbeat timeout (default 10s) → Kafka thinks consumer dead
    → Rebalance triggered → all consumers restart → chaos
    
  Right size:
    max-poll-records × processing_time_per_record < max.poll.interval.ms
    e.g., 10 records × 200ms = 2s < 5min (default interval) ✅
```

### RabbitMQ — Prefetch Count as Back-pressure

```
RabbitMQ PUSHES messages to consumers (unlike Kafka's pull).
Without limits, broker sends ALL queued messages to consumer instantly.

  Queue: 100,000 messages
  Consumer starts
  RabbitMQ: pushes all 100,000 immediately
  Consumer memory: 💥

Fix: prefetch count (QoS):
  consumer.basicQos(prefetchCount = 10)

  Meaning:
    "Send me max 10 unacknowledged messages at a time.
     Don't send more until I ACK some."

  ┌──────────────────────────────────────────────────────┐
  │  RabbitMQ Queue                                     │
  │  [msg1][msg2]...[msg100000]                         │
  │                                                      │
  │  prefetch = 10:                                      │
  │  → sends msg1-10 to consumer                        │
  │  → STOPS. Waits for ACKs.                           │
  │  Consumer processes msg1 → ACKs                     │
  │  → RabbitMQ sends msg11                             │
  │  → always max 10 in-flight                          │
  └──────────────────────────────────────────────────────┘

  prefetchCount = 1:   safest, slowest (one at a time)
  prefetchCount = 10:  balanced
  prefetchCount = 0:   unlimited — NO back-pressure ❌
```

```java
// ShopSphere notification-service RabbitMQ consumer:
@RabbitListener(
    queues = "notification.queue",
    containerFactory = "rabbitListenerContainerFactory"
)
public void handleNotification(NotificationEvent event) {
    emailService.sendEmail(event);  // takes 100ms
    // ACK sent automatically after method returns (if no exception)
    // RabbitMQ sends next message only after ACK
}

// Config:
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory =
        new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(10);           // back-pressure: max 10 in-flight
    factory.setAcknowledgeMode(AcknowledgeMode.AUTO);
    return factory;
}
```

---

## Reactive Streams — Back-pressure as First-Class Citizen

Spring WebFlux and Project Reactor make back-pressure a core primitive.

```
Reactive Streams protocol:

  Publisher (produces data)
  Subscriber (consumes data)
  
  Without back-pressure (push model):
    Publisher: "Here's 10,000 items!" → overwhelms Subscriber
  
  With Reactive Streams back-pressure (pull model):
    Subscriber sends REQUEST(n): "I can handle n items right now"
    Publisher sends EXACTLY n items — no more
    Subscriber processes, sends REQUEST(n) again when ready
    
  Publisher never sends more than Subscriber requested.
  Back-pressure built into the protocol.
```

```java
// Project Reactor back-pressure in action:
Flux.range(1, 1_000_000)               // publisher: 1M items
    .onBackpressureBuffer(1000)         // buffer 1000 if subscriber slow
    .publishOn(Schedulers.boundedElastic())
    .subscribe(
        item -> {
            processSlowly(item);        // takes 10ms each
        },
        error -> log.error("Error", error),
        () -> log.info("Complete")
    );

// Subscriber processes at its own pace.
// Buffer absorbs burst up to 1000.
// If buffer fills → onBackpressureBuffer drops or errors — explicit signal.
```

```
Strategies when buffer fills (overflow):
  onBackpressureBuffer()    → buffer unbounded (careful — OOM risk)
  onBackpressureBuffer(N)   → buffer N items, then error
  onBackpressureDrop()      → drop new items silently (data loss)
  onBackpressureLatest()    → keep only most recent item
  onBackpressureError()     → emit error signal to subscriber
```

---

## Back-pressure Failure — What Happens Without It

```
Real incident pattern (ShopSphere Flash Sale scenario):

t=0s:    Flash sale starts
         10,000 orders/sec hitting order-service

t=1s:    order-service → publishes to Kafka: order-placed (10,000/sec)

t=2s:    notification-service consumes Kafka:
         Processing rate: 500 emails/sec
         Kafka lag: growing at 9,500/sec

t=10s:   Kafka lag: 95,000 messages
         notification-service has no back-pressure signal to order-service
         order-service keeps producing at 10,000/sec

t=60s:   Kafka lag: 570,000 messages
         notification-service thread pool saturated
         Email provider (SendGrid) rate limit hit → 429s
         notification-service starts failing
         Unacknowledged messages → RabbitMQ requeues → redelivery
         → notification-service gets DOUBLE the messages

t=120s:  notification-service OOM crash
         All unacked messages requeued → burst of redeliveries
         → notification-service pod restarts into same storm

Without back-pressure: cascade. With back-pressure:
  prefetch=10 → notification-service only takes what it can handle
  Kafka lag grows (normal — Kafka is designed for this)
  order-service unaffected
  notification-service stable at 500 emails/sec
  Lag drains gradually after flash sale ends
```

---

## Back-pressure Patterns — Summary Table

```
┌───────────────────────┬──────────────────────────────┬─────────────────────┐
│  Pattern              │  Mechanism                   │  ShopSphere Usage   │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  Bounded queue        │  CallerRunsPolicy blocks      │  Thread pools in    │
│  (JVM level)          │  producer when full           │  all services       │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  HTTP 429/503         │  Explicit signal to caller    │  Gateway rate limit │
│                       │  + Retry-After header         │  response           │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  Circuit Breaker      │  Stop calling failing service │  order→payment,     │
│                       │  Give it recovery time        │  Resilience4j       │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  Kafka consumer lag   │  Pull model — consumer sets   │  notification-svc   │
│                       │  its own pace                 │  KEDA autoscaling   │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  RabbitMQ prefetch    │  Max N unacked messages       │  notification-svc   │
│                       │  Broker waits for ACKs        │  prefetch=10        │
├───────────────────────┼──────────────────────────────┼─────────────────────┤
│  Reactive Streams     │  Subscriber requests N items  │  WebFlux endpoints  │
│                       │  Publisher sends exactly N    │  if added later     │
└───────────────────────┴──────────────────────────────┴─────────────────────┘
```

---

## Interview Angles

**Q: What is back-pressure and why is it needed?**
> Back-pressure is the mechanism by which a slow or overwhelmed consumer signals upstream producers to slow down. Without it, a fast producer will flood a slow consumer — filling queues, exhausting memory, and eventually crashing the consumer. Back-pressure makes the system self-regulating, matching production rate to consumption capacity.

**Q: How does Kafka handle back-pressure?**
> Kafka's pull model is inherently back-pressure friendly. Consumers fetch at their own pace — they request batches when ready, not when the broker decides to push. Consumer lag (the gap between latest offset and consumer offset) is the natural back-pressure signal. KEDA can watch this lag and autoscale consumers when it grows, automatically adding capacity in response to overload.

**Q: What's the difference between circuit breaker and back-pressure?**
> They're related but different. Back-pressure is the general concept of a consumer signaling to slow down. Circuit breaker is a specific pattern implementing back-pressure at the service-to-service level — when failure rate crosses a threshold, the caller stops sending requests entirely, giving the downstream service recovery time. The CB is the upstream's automated response to the downstream's distress.

**Q: What happens if RabbitMQ prefetchCount = 0?**
> The broker pushes all queued messages to the consumer immediately with no limit. The consumer's in-memory buffer fills instantly, memory exhausts, and the consumer crashes. All those unacknowledged messages get requeued and delivered again at restart — making the next boot even worse. Always set a finite prefetch count.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  What        │ Consumer signals upstream: "slow down, I'm full"      │
├──────────────────────────────────────────────────────────────────────┤
│  Without it  │ Fast producer + slow consumer = queue overflow,       │
│              │ OOM, crash, retry storm, cascading failure            │
├──────────────────────────────────────────────────────────────────────┤
│  JVM level   │ Bounded queue + CallerRunsPolicy                      │
│  HTTP level  │ 429/503 + Retry-After + Circuit Breaker               │
│  Kafka       │ Pull model, consumer lag metric, KEDA autoscale       │
│  RabbitMQ    │ prefetchCount = N (max N unacked messages)            │
│  Reactive    │ REQUEST(n) protocol — publisher sends exactly n       │
├──────────────────────────────────────────────────────────────────────┤
│  ShopSphere  │ Bounded thread pools everywhere. prefetch=10 on       │
│              │ RabbitMQ. Kafka pull pace. CB on payment/external.    │
│              │ KEDA on notification-svc lag.                         │
└──────────────────────────────────────────────────────────────────────┘
```

---

Seven down, one to go. **Topic 8 — Autoscaling**: horizontal pod autoscaling internals, what metrics trigger scaling, cooldown periods, and the subtle ways autoscaling can go wrong. Ready?
