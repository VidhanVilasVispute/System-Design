# Stage 7 — Distributed Systems
## Topic 9: Saga Pattern

---

## The Problem 2PC Left Us With

From Topic 8 — 2PC gives strong consistency but:

```
❌ Blocks on coordinator failure
❌ Holds locks across services
❌ Tight coupling between services
❌ Low throughput under contention
❌ Violates microservice independence
```

We need distributed coordination **without** distributed locks.

The Saga pattern is the answer. Introduced by Hector Garcia-Molina in 1987, it was originally designed for long-running database transactions. The microservices world rediscovered it as the standard solution for multi-service coordination.

---

## The Core Idea

Instead of one big atomic transaction — **break it into a sequence of local transactions**, each in its own service, each publishing an event when complete.

```
2PC thinking:
  "Lock everything, do everything atomically, unlock."

Saga thinking:
  "Do step 1 locally. If it succeeds, trigger step 2.
   If step 2 fails, undo step 1 with a compensating transaction."
```

A Saga is:

```
T1 → T2 → T3 → T4 → ... → Tn   (happy path)

If Tk fails:
  run compensating transactions in reverse:
  C(k-1) → C(k-2) → ... → C1   (rollback path)
```

Each `Ti` is a local transaction. Each `Ci` is its compensating transaction — a business-level undo.

---

## Compensating Transactions — The Key Concept

A compensating transaction is **not a database ROLLBACK**. It's a new forward transaction that semantically undoes a previous one.

```
Original transaction T:        Compensating transaction C(T):
────────────────────────────────────────────────────────────────
INSERT order (status=PLACED)   UPDATE order SET status=CANCELLED
DECREMENT stock by 1           INCREMENT stock by 1
CHARGE payment $99             REFUND payment $99
SEND confirmation email        SEND cancellation email
```

Critical distinction:

```
Database ROLLBACK:
  Pretends the transaction never happened.
  No trace in the log. Works inside one DB.

Compensating Transaction:
  Acknowledges it happened AND creates a new record undoing it.
  Audit trail preserved. Works across services.
  Customer sees: "Order placed" then "Order cancelled + refund issued"
```

Not every operation needs compensation:

```
Compensatable:    payment charge → refund
                  stock decrement → stock increment
                  order creation → order cancellation

NOT compensatable: email sent → can't unsend
                   SMS sent → can't unsend
                   third-party audit log → can't delete
```

For non-compensatable steps → accept the side effect, or delay them until the saga is confirmed complete.

---

## Two Implementation Styles

```
Saga Implementation
        │
        ├── Choreography  →  services talk to each other via events
        │                    no central coordinator
        │
        └── Orchestration →  central orchestrator tells each service
                             what to do next
```

---

## Style 1: Choreography

Each service publishes events. Other services listen and react. No one is "in charge."

```
Order Placement Saga — Choreography Style:

order-service          inventory-service       payment-service
      │                        │                      │
      │ place order             │                      │
      │ save (PENDING)          │                      │
      │──OrderCreated──────────▶│                      │
      │                         │ check + reserve      │
      │                         │ stock                │
      │                         │──StockReserved──────▶│
      │                         │                      │ charge card
      │                         │                      │
      │◀────────────────────────│◀──PaymentProcessed───│
      │ update status           │                      │
      │ → CONFIRMED             │                      │
      │                         │ confirm reservation  │
```

**Failure path — payment fails:**

```
order-service          inventory-service       payment-service
      │                        │                      │
      │──OrderCreated──────────▶│                      │
      │                         │──StockReserved──────▶│
      │                         │                      │ card declined
      │                         │◀──PaymentFailed──────│
      │                         │ release reservation  │
      │◀──StockReleased─────────│                      │
      │ update status           │                      │
      │ → CANCELLED             │                      │
```

### Choreography Implementation in ShopSphere

```java
// order-service: start the saga
@Service
public class OrderService {

    @Transactional
    public Order placeOrder(OrderRequest req) {
        Order order = Order.builder()
            .userId(req.getUserId())
            .productId(req.getProductId())
            .status(OrderStatus.PENDING)
            .build();
        orderRepository.save(order);

        // publish event — saga begins
        eventPublisher.publish(new OrderCreatedEvent(
            order.getId(),
            order.getProductId(),
            order.getUserId(),
            order.getAmount()
        ));

        return order;
    }

    // listen for saga outcomes
    @KafkaListener(topics = "payment-events")
    public void onPaymentResult(PaymentEvent event) {
        Order order = orderRepository.findById(event.getOrderId());

        if (event instanceof PaymentProcessedEvent) {
            order.setStatus(OrderStatus.CONFIRMED);
        } else if (event instanceof PaymentFailedEvent) {
            order.setStatus(OrderStatus.CANCELLED);
            // compensation already handled by inventory-service
        }
        orderRepository.save(order);
    }
}
```

```java
// inventory-service: listens, acts, publishes
@Service
public class InventoryService {

    @KafkaListener(topics = "order-events")
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        Product product = productRepository
            .findByIdWithLock(event.getProductId());  // SELECT FOR UPDATE

        if (product.getStock() > 0) {
            product.setStock(product.getStock() - 1);
            product.setReservedStock(product.getReservedStock() + 1);
            productRepository.save(product);
            eventPublisher.publish(new StockReservedEvent(event.getOrderId()));
        } else {
            eventPublisher.publish(new StockUnavailableEvent(event.getOrderId()));
        }
    }

    // compensating transaction — payment failed, release reservation
    @KafkaListener(topics = "payment-events")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        Product product = productRepository
            .findByOrderReservation(event.getOrderId());
        product.setStock(product.getStock() + 1);         // undo decrement
        product.setReservedStock(product.getReservedStock() - 1);
        productRepository.save(product);
        eventPublisher.publish(new StockReleasedEvent(event.getOrderId()));
    }
}
```

### Choreography — Pros and Cons

```
Pros:
  ✅ No central coordinator — no SPOF
  ✅ Loose coupling — services only know about events
  ✅ Easy to add new steps — subscribe to existing events
  ✅ Scales well — each service handles its own load

Cons:
  ❌ Hard to track saga state — "where is this order in the flow?"
  ❌ Cyclic dependencies risk — service A listens to B, B listens to A
  ❌ Testing is hard — must simulate full event chains
  ❌ Difficult to visualize the overall flow
  ❌ Compensation logic scattered across all services
```

---

## Style 2: Orchestration

A central **orchestrator** — a separate service or state machine — explicitly tells each service what to do next. Services execute and report back.

```
Order Placement Saga — Orchestration Style:

         OrderSagaOrchestrator
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
inventory   payment      order
 service    service      service

Step 1: Orchestrator → inventory-service: "Reserve stock for order X"
Step 2: inventory-service → Orchestrator: "Reserved ✅"
Step 3: Orchestrator → payment-service:  "Charge $99 for order X"
Step 4: payment-service → Orchestrator:  "Charged ✅"
Step 5: Orchestrator → order-service:    "Confirm order X"
Step 6: order-service → Orchestrator:    "Confirmed ✅"
Step 7: Orchestrator: SAGA COMPLETE ✅
```

**Failure path:**

```
Step 1: Orchestrator → inventory-service: "Reserve stock"
Step 2: inventory-service → Orchestrator: "Reserved ✅"
Step 3: Orchestrator → payment-service:  "Charge $99"
Step 4: payment-service → Orchestrator:  "FAILED ❌"

Orchestrator triggers compensations in reverse:
Step 5: Orchestrator → inventory-service: "Release reservation"  (C1)
Step 6: inventory-service → Orchestrator: "Released ✅"
Step 7: Orchestrator → order-service:    "Cancel order"          (C0)
Step 8: Orchestrator: SAGA ROLLED BACK ✅
```

### Orchestration State Machine

The orchestrator is a **state machine** — each order saga has a current state:

```
States:
──────────────────────────────────────────────────────────────────
STARTED
  │
  ▼
STOCK_RESERVING ──(failed)──▶ CANCELLED
  │
  ▼(success)
PAYMENT_PROCESSING ──(failed)──▶ COMPENSATING_STOCK ──▶ CANCELLED
  │
  ▼(success)
ORDER_CONFIRMING
  │
  ▼
COMPLETED
──────────────────────────────────────────────────────────────────
```

### Orchestration Implementation in ShopSphere

```java
// Saga state — persisted to DB
@Entity
@Table(name = "order_saga")
public class OrderSaga {

    @Id
    private UUID sagaId;
    private UUID orderId;

    @Enumerated(EnumType.STRING)
    private SagaStatus status;  // STARTED, STOCK_RESERVING, etc.

    private String failureReason;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

```java
// Orchestrator service
@Service
@Slf4j
public class OrderSagaOrchestrator {

    @Autowired OrderSagaRepository sagaRepository;
    @Autowired KafkaTemplate<String, Object> kafka;

    // Step 1: start saga
    @Transactional
    public void startSaga(Order order) {
        OrderSaga saga = OrderSaga.builder()
            .sagaId(UUID.randomUUID())
            .orderId(order.getId())
            .status(SagaStatus.STOCK_RESERVING)
            .build();
        sagaRepository.save(saga);

        kafka.send("inventory-commands",
            new ReserveStockCommand(order.getId(), order.getProductId()));
    }

    // Step 2: stock reserved → charge payment
    @KafkaListener(topics = "inventory-replies")
    @Transactional
    public void onInventoryReply(InventoryReply reply) {
        OrderSaga saga = sagaRepository.findByOrderId(reply.getOrderId());

        if (reply.isSuccess()) {
            saga.setStatus(SagaStatus.PAYMENT_PROCESSING);
            sagaRepository.save(saga);
            kafka.send("payment-commands",
                new ProcessPaymentCommand(reply.getOrderId(), reply.getAmount()));
        } else {
            // stock unavailable — cancel immediately, no compensation needed
            saga.setStatus(SagaStatus.CANCELLED);
            saga.setFailureReason("Stock unavailable");
            sagaRepository.save(saga);
            kafka.send("order-commands",
                new CancelOrderCommand(reply.getOrderId(), "Stock unavailable"));
        }
    }

    // Step 3: payment result → confirm or compensate
    @KafkaListener(topics = "payment-replies")
    @Transactional
    public void onPaymentReply(PaymentReply reply) {
        OrderSaga saga = sagaRepository.findByOrderId(reply.getOrderId());

        if (reply.isSuccess()) {
            saga.setStatus(SagaStatus.COMPLETED);
            sagaRepository.save(saga);
            kafka.send("order-commands",
                new ConfirmOrderCommand(reply.getOrderId()));
        } else {
            // payment failed → compensate stock reservation
            saga.setStatus(SagaStatus.COMPENSATING_STOCK);
            saga.setFailureReason("Payment failed: " + reply.getReason());
            sagaRepository.save(saga);
            kafka.send("inventory-commands",
                new ReleaseStockCommand(reply.getOrderId()));  // compensating tx
        }
    }

    // Step 4: compensation complete → cancel order
    @KafkaListener(topics = "inventory-compensation-replies")
    @Transactional
    public void onStockReleased(StockReleasedEvent event) {
        OrderSaga saga = sagaRepository.findByOrderId(event.getOrderId());
        saga.setStatus(SagaStatus.CANCELLED);
        sagaRepository.save(saga);
        kafka.send("order-commands",
            new CancelOrderCommand(event.getOrderId(), saga.getFailureReason()));
    }
}
```

---

## Orchestration vs Choreography — When to Use Which

```
Dimension               Choreography          Orchestration
──────────────────────────────────────────────────────────────────
Coupling                Loose (events only)   Tighter (orchestrator knows all)
Observability           Hard — state scattered Easy — state in orchestrator
Complexity location     In each service       In orchestrator
Single point of failure No SPOF               Orchestrator can be SPOF*
Flow visibility         Implicit              Explicit state machine
Adding new steps        Easy (new subscriber) Must update orchestrator
Testing                 Hard (event chains)   Easier (test orchestrator)
Good for                Simple, linear flows  Complex flows with many branches
──────────────────────────────────────────────────────────────────
* Orchestrator SPOF solved by making it stateful + replicated (Kafka + DB)
```

**Pragmatic guidance for ShopSphere:**

```
Use Choreography when:
  → 2–3 services involved
  → Linear happy path, simple failure cases
  → Teams own their services end-to-end
  → Example: notification-service reacting to order events

Use Orchestration when:
  → 4+ services involved
  → Multiple branching failure scenarios
  → Need full audit trail of saga progress
  → Example: order placement touching inventory + payment + shipping + loyalty
```

---

## The Isolation Problem — Sagas Are NOT ACID

This is the most important thing to understand about Sagas:

```
Saga gives you ACD — not ACID.

A — Atomicity:    Yes (eventual — compensations ensure it)
C — Consistency:  Yes (business-rule consistency at each step)
I — Isolation:    ❌ NO
D — Durability:   Yes (each local transaction is durable)
```

**The isolation gap:**

```
Order A saga starts:
  Step 1: reserve stock (stock: 10 → 9)
  Step 2: process payment ← running...

MEANWHILE:
  Order B saga starts:
  Step 1: reserve stock (stock: 9 → 8)  ← sees Order A's intermediate state!

Order A payment FAILS:
  Compensate: release stock (stock: 8 → 9)

Final state: stock = 9 ✅ (correct)

But Order B saw stock = 8 during Order A's saga.
It made a decision based on intermediate state it shouldn't have seen.
That's a DIRTY READ across sagas.
```

### Countermeasures for Isolation Problems

```
1. Semantic Locking:
   Mark records being modified by a saga with a flag.
   Other transactions see the flag and either wait or fail-fast.

   order.setStatus(PENDING_RESERVATION) ← semantic lock
   inventory checks: if order has PENDING_RESERVATION → handle accordingly

2. Commutative Operations:
   Design operations so order doesn't matter.
   "Decrement by 1" is commutative — result is same regardless of order.

3. Pessimistic Views:
   Read minimum data. Don't show data that's in-flight.
   search-service: don't show products with PENDING orders as "in stock"

4. Version/ETag checks:
   Read stock level + version. Write only if version unchanged.
   If version changed (another saga modified it) → retry or fail.
   This is optimistic locking — PostgreSQL supports it natively.
```

---

## Saga vs 2PC — Final Comparison

```
Property              2PC                      Saga
──────────────────────────────────────────────────────────────────
Atomicity             Strict (ACID)            Eventual (compensations)
Isolation             Full (locks)             None (dirty reads possible)
Blocking              Yes — on failure         No — async steps
Throughput            Low (locking)            High (no cross-service locks)
Latency               High (sync round trips)  Low (async)
Failure handling      Coordinator decision     Compensating transactions
Coupling              Tight (sync locks)       Loose (async events)
Observability         Implicit in protocol     Explicit (orchestration)
Use case              Financial DBs, Spanner   Microservices everywhere
```

---

## Interview Angles 🎯

**Q: What is a Saga and why do microservices use it instead of 2PC?**
> A Saga is a sequence of local transactions across services, where each step publishes an event triggering the next. If any step fails, compensating transactions undo previous steps. Microservices use it instead of 2PC because 2PC requires cross-service locks that block on failure, creating tight coupling and availability problems. Sagas are async, loosely coupled, and don't hold locks between steps.

**Q: What's the difference between choreography and orchestration?**
> In choreography, each service reacts to events published by other services — there's no central coordinator, and the flow emerges from the interactions. In orchestration, a central orchestrator explicitly commands each service and manages the saga state machine. Choreography is simpler and more decoupled but hard to trace. Orchestration is easier to observe and test but introduces an orchestrator as a potential bottleneck.

**Q: What is a compensating transaction and how does it differ from a database rollback?**
> A compensating transaction is a new forward transaction that semantically undoes a previous one. A database rollback pretends the original transaction never happened and leaves no trace. Compensating transactions are necessary in sagas because the original local transactions have already committed — you can't roll them back. The compensation creates a new audit record: "this happened, then this was undone."

**Q: Sagas don't provide isolation — what problems does that cause and how do you handle it?**
> Without isolation, one saga can read intermediate state from another saga that hasn't completed — a dirty read across services. For example, a flash sale might oversell if two sagas both read "1 item in stock" before either commits. Countermeasures include semantic locking (flagging in-flight records), optimistic locking with version checks, pessimistic views (don't show in-flight inventory as available), and designing commutative operations.

**Q: How would you implement a saga for ShopSphere's order placement?**
> I'd use orchestration since it involves inventory, payment, and order services with multiple failure branches. The order-service hosts an OrderSagaOrchestrator with a persisted state machine. It sends commands to each service via Kafka and listens for replies. On payment failure, it sends a compensating ReleaseStockCommand to inventory, then cancels the order. The saga state is stored in a DB table so it survives orchestrator restarts.

---

Say **next** for **Topic 10: Event Sourcing** — store the log of changes, not just current state, and replay to reconstruct anything 🚀
