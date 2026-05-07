# Stage 2 — Topic 5: Synchronous vs Asynchronous Communication

## Theory

Every time two services need to communicate, you face a fundamental architectural decision — does the caller **wait** for the result, or does it **fire and move on**?

This is the synchronous vs asynchronous decision. It is not just a technical choice — it determines your system's resilience, scalability, user experience, and operational complexity. Getting it wrong at the architecture stage is expensive to fix later.

**Synchronous communication:**
The caller sends a request and **blocks** — it waits, doing nothing else, until it receives a response. The caller and receiver are **temporally coupled** — they must both be available at the same moment.

**Asynchronous communication:**
The caller sends a message and **continues immediately** — it does not wait for the receiver to process it. The caller and receiver are **temporally decoupled** — they do not need to be available at the same time.

```
Synchronous — phone call:
  You call someone. You both must be present simultaneously.
  If they don't pick up, you're stuck.
  You wait for their answer before doing anything else.

Asynchronous — text message:
  You send a message and put your phone down.
  They read and respond when they're available.
  You've already moved on to other things.
```

---

## Internals — Synchronous Communication

### How It Works

```
Service A                    Service B
    |                            |
    |--- HTTP/gRPC request ----->|
    |                            | (processing...)
    |     (A is blocked,         | (processing...)
    |      doing nothing)        | (processing...)
    |                            |
    |<-- response ---------------|
    |
    | (A continues)
```

**What "blocked" means at the thread level:**

```java
// Service A — synchronous REST call
public OrderResponse createOrder(CreateOrderRequest request) {
    
    // Thread is BLOCKED here — sitting idle waiting for Payment Service
    PaymentResponse payment = paymentClient.processPayment(request.getPaymentDetails());
    // ↑ This thread cannot do anything else until payment responds
    
    // Thread is BLOCKED here waiting for Inventory Service  
    InventoryResponse inventory = inventoryClient.reserveStock(request.getItems());
    
    // Only now does execution continue
    return buildOrderResponse(payment, inventory);
}
```

If `paymentClient.processPayment()` takes 2 seconds, that thread is idle for 2 seconds. In a thread-per-request model with 200 threads, if 200 requests are all waiting on slow downstream services — your service is completely unresponsive to new requests even though it is doing zero work.

### The Synchronous Call Chain Problem

```
Client → API Gateway → Order Service → Payment Service → Stripe API
                                    ↘ Inventory Service → DB
                                    ↘ Notification Service → SendGrid

If any service in the chain is slow or down:
  - Every caller above it stalls
  - Threads pile up across the entire chain
  - Memory fills with blocked threads
  - The entire system degrades together
  - This is called a cascading failure
```

**Latency compounds in synchronous chains:**
```
Sequential synchronous calls:
  Order validation:    50ms
  Inventory check:    100ms
  Payment processing: 800ms
  Fraud check:        200ms
  ─────────────────────────
  Total user wait:   1150ms

One service gets slow (Stripe congestion):
  Payment processing: 3000ms
  Total user wait:    3350ms  ← user experience ruined by one downstream service
```

### When Synchronous Is Correct

Despite its downsides, synchronous communication is the right choice when:

```
1. You need the result to continue:
   Cannot confirm an order without knowing if payment succeeded.
   Cannot serve a product page without fetching product data.

2. The operation must complete before responding to the user:
   Login — user needs to know if credentials are valid right now.
   Search — user is waiting on the results page.

3. The operation is fast and reliable:
   Redis cache lookup: ~1ms — no reason to make this async.
   In-memory computation: microseconds.

4. Strong consistency is required:
   Checking available seats before booking a flight.
   Verifying account balance before a withdrawal.
```

---

## Internals — Asynchronous Communication

### How It Works

```
Service A                  Message Broker           Service B
    |                           |                       |
    |--- publish message ------>|                       |
    |                           |                       |
    | (A continues immediately) |--- deliver message -->|
    |                           |                       | (processing...)
    |                           |                       | (processing...)
    |                           |                       | (done)
```

Service A never knows when Service B processed the message. It does not know if Service B succeeded or failed. It just knows the message was accepted by the broker.

### The Three Async Patterns

**Pattern 1 — Fire and Forget:**
```
Caller publishes event → never expects a response

Example in ShopSphere:
  Order Service publishes OrderConfirmed event
  → Notification Service sends confirmation email (fire and forget)
  → Analytics Service records the sale (fire and forget)
  → Loyalty Service awards points (fire and forget)

Order Service does not care when or whether these happen.
If Notification Service is down, the email is delayed — not lost.
The order is already confirmed and the user is already happy.
```

**Pattern 2 — Async Request/Reply:**
```
Caller publishes request with a correlation ID and reply-to queue
Callee processes and publishes response to reply-to queue
Caller consumes from its reply queue matching by correlation ID

Producer                Broker                  Consumer
   |                      |                         |
   |-- request            |                         |
   |   correlationId: 123 |                         |
   |   replyTo: q-A   --->|--- deliver request ---->|
   |                      |                    (processing...)
   |                      |<-- response ------------|
   |                      |    correlationId: 123
   |<-- poll q-A ---------|
   | (match correlationId 123, resume)
```

More complex but enables async without losing the response. Used when you need the result eventually but don't need to block waiting.

**Pattern 3 — Event-Driven (Pub/Sub):**
```
Publisher emits events without knowing who is listening.
Multiple subscribers react independently.

ShopSphere — OrderConfirmed event:
  Publisher:  Order Service emits { orderId, userId, items, total }
  
  Subscriber 1: Notification Service  → sends confirmation email
  Subscriber 2: Inventory Service     → decrements stock permanently
  Subscriber 3: Analytics Service     → records revenue
  Subscriber 4: Loyalty Service       → awards points
  Subscriber 5: Recommendation Engine → updates user purchase history

Adding a new subscriber requires zero changes to Order Service.
Each subscriber processes independently, at its own pace.
```

---

## The Real Trade-offs

### Coupling

```
Synchronous — temporal coupling:
  Order Service REQUIRES Payment Service to be UP right now.
  Payment Service deploys a breaking change → Order Service breaks immediately.
  
Asynchronous — decoupled:
  Order Service requires the MESSAGE BROKER to be up.
  Payment Service can be down, deploying, or slow — messages queue up.
  When Payment Service recovers, it processes the backlog.
  Order Service never noticed.
```

### Consistency

```
Synchronous:
  Payment processes → Order confirmed in same transaction flow
  Immediate consistency — user sees confirmed order right away
  
Asynchronous:
  Order accepted → payment processes in background → order updated
  Eventual consistency — user might briefly see "PENDING" before "CONFIRMED"
  This requires UI design to handle in-progress states gracefully
```

### Failure Handling

```
Synchronous failure:
  Payment Service returns 500
  → Order Service must handle it RIGHT NOW
  → Return error to user immediately
  → Simple but user-facing

Asynchronous failure:
  Payment processing fails
  → Message goes to dead letter queue
  → Retry automatically with backoff
  → Alert on-call engineer if retries exhausted
  → User does not see failure immediately
  → More resilient but more complex to debug
```

### Observability

```
Synchronous — easy to trace:
  Request A → Service B → Service C
  One trace ID, linear call chain
  Easy to see exactly where latency comes from

Asynchronous — harder to trace:
  Event published at t=0 → processed at t=5s → downstream effect at t=8s
  Requires correlation IDs threaded through every message
  Requires distributed tracing across async boundaries
  Much harder to answer "why did this order take 30 seconds?"
```

---

## Hybrid Architecture — The Right Answer

Real systems use both. The skill is knowing which to use where.

```
ShopSphere Order Creation — hybrid flow:

SYNCHRONOUS (user is waiting):
  1. Validate request input          — must know immediately if invalid
  2. Check item availability         — must know if in stock before charging
  3. Process payment                 — must know if payment succeeded
  4. Create order record             — must have order ID to return to user
  5. Return 201 Created to user      ← user gets response here, ~300ms total

ASYNCHRONOUS (user doesn't need to wait):
  6. Send confirmation email         — async via RabbitMQ → Notification Service
  7. Decrement inventory permanently — async event → Inventory Service
  8. Award loyalty points            — async event → Loyalty Service
  9. Update recommendations          — async event → Recommendation Engine
  10. Record analytics               — async event → Analytics Service
```

The user gets a fast response (300ms for the critical path). The non-critical work happens in the background without affecting the user's wait time.

---

## Identifying Synchronous vs Asynchronous in System Design Interviews

Use this decision framework:

```
Does the caller need the result to continue its own logic?
  YES → Synchronous
  NO  → Consider async

Does the user need to see the result immediately?
  YES → Synchronous (at least for that part)
  NO  → Async is fine

Is strict consistency required right now?
  YES → Synchronous
  NO  → Async with eventual consistency

Can the operation be retried safely if it fails?
  YES → Async with retry is good
  NO  → Synchronous with immediate error handling

Is the downstream operation slow or unreliable?
  YES → Async protects you from its failures
  NO  → Synchronous is simpler
```

---

## Real-World Example — ShopSphere Event Flow

**Synchronous path — order placement critical path:**
```java
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
        @RequestBody CreateOrderRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // All synchronous — user waits for all of this
    validateRequest(request);                              // ~5ms
    StockResponse stock = inventoryClient.checkStock();    // ~50ms (gRPC)
    PaymentResponse payment = paymentClient.charge();      // ~800ms (Stripe)
    Order order = orderRepository.save(buildOrder());      // ~10ms
    
    // Now publish events — fire and forget
    // User does NOT wait for any of this
    kafkaTemplate.send("order.confirmed", OrderConfirmedEvent.builder()
        .orderId(order.getId())
        .userId(order.getUserId())
        .items(order.getItems())
        .total(order.getTotal())
        .build());
    
    // Return to user immediately after saving
    return ResponseEntity.status(201).body(buildResponse(order));
}
```

**Asynchronous consumers — each independent:**
```java
// Notification Service — listens to order.confirmed
@KafkaListener(topics = "order.confirmed")
public void handleOrderConfirmed(OrderConfirmedEvent event) {
    emailService.sendConfirmation(event.getUserId(), event.getOrderId());
    // If this fails, Kafka retries automatically
    // Order Service and user are completely unaffected
}

// Loyalty Service — listens to same topic independently
@KafkaListener(topics = "order.confirmed")
public void awardPoints(OrderConfirmedEvent event) {
    loyaltyService.award(event.getUserId(), event.getTotal());
}

// Analytics Service — listens to same topic independently  
@KafkaListener(topics = "order.confirmed")
public void recordSale(OrderConfirmedEvent event) {
    analyticsService.record(event);
}
```

All three consumers are completely independent — they each have their own consumer group, their own offset, their own retry logic. A failure in Loyalty Service does not affect Notification Service or Analytics Service.

---

## Interview Q&A

**Q: What is the difference between synchronous and asynchronous communication?**
In synchronous communication the caller blocks and waits for the response before continuing — both services must be available simultaneously. In asynchronous communication the caller publishes a message and continues immediately — the receiver processes it independently, possibly much later. Synchronous communication is simpler and gives immediate consistency. Asynchronous decouples services, improves resilience, and allows the critical path to complete without waiting for non-critical work.

**Q: When would you choose async over sync in a microservices system?**
When the caller does not need the result to continue its own logic, when the operation is non-critical to the immediate user response, when you want to protect the caller from downstream slowness or failures, and when the work can be retried safely. Classic examples are sending emails, updating analytics, awarding loyalty points, and updating search indexes — all triggered by an order event but none required for the user to receive their order confirmation.

**Q: What is temporal coupling and why is it a problem?**
Temporal coupling means two services must be available at the same time for communication to succeed. Synchronous HTTP calls are temporally coupled — if the downstream service is deploying, slow, or crashed, the upstream caller fails immediately. Asynchronous messaging removes temporal coupling — the message broker accepts the message even if the consumer is unavailable, and the consumer processes it when it recovers. This makes the system dramatically more resilient to partial failures.

**Q: How do you maintain data consistency with asynchronous communication?**
Asynchronous systems achieve eventual consistency — all services will be consistent eventually, but there may be a brief window of inconsistency. The pattern is to complete the critical synchronous operations first — validate, charge, persist — then publish events for the rest. The UI must be designed to handle intermediate states gracefully — showing "Processing" while async work completes. For operations that require strong consistency across services, you use the Saga pattern with compensating transactions rather than a distributed synchronous transaction.

**Q: How do you trace a request across asynchronous boundaries?**
Propagate a correlation ID through every message. When an order is created it gets a trace ID. Every event published from that order includes the same trace ID in the message headers. Every consumer that processes the event logs the trace ID and propagates it to any further messages or service calls it makes. With a distributed tracing system like Jaeger or Zipkin, you can reconstruct the complete causal chain of an operation even when it spans multiple async hops across services and time.

---

Say **"next"** when ready for Topic 6 — Message-Based Communication.
