# Stage 7 — Distributed Systems
## Topic 11: CQRS (Command Query Responsibility Segregation)

---

## The Problem With a Single Model

In a traditional application, one model handles everything:

```java
// One repository, does everything
public interface OrderRepository extends JpaRepository<Order, UUID> {
    Order findById(UUID id);                          // simple read
    List<Order> findByUserId(UUID userId);            // user history
    List<Order> findByStatusAndCreatedAtBetween(...); // admin dashboard
    Page<Order> findAllWithProductDetails(...);       // complex join query
    void save(Order order);                           // write
    void delete(Order order);                         // write
}
```

This works fine at small scale. Problems emerge when:

```
Write side needs:
  → Strong consistency
  → Transactional integrity
  → Domain validation
  → Optimistic locking
  → Normalized schema (avoid update anomalies)

Read side needs:
  → Speed (sub-10ms response)
  → Denormalized joins (one query, no N+1)
  → Flexible filtering, sorting, pagination
  → Different shapes for different consumers
    (mobile needs less data than admin dashboard)
  → Can tolerate slight staleness
```

These needs **conflict**. A normalized schema optimized for writes is slow to query. A denormalized schema fast to read is painful to keep consistent on writes.

**CQRS** resolves this conflict by splitting them entirely.

---

## The Core Idea

```
CQRS = Command Query Responsibility Segregation

Commands  → change state    → go to the WRITE model
Queries   → read state      → go to the READ model

One model handles one responsibility. Never both.
```

```
                    ┌─────────────────────────────────┐
                    │         Your Application         │
                    └─────────────┬───────────┬────────┘
                                  │           │
                          Commands│           │Queries
                                  ▼           ▼
                    ┌─────────────────┐   ┌──────────────────┐
                    │   WRITE MODEL   │   │   READ MODEL      │
                    │                 │   │                    │
                    │ Normalized DB   │   │ Denormalized DB    │
                    │ Domain logic    │   │ Optimized views    │
                    │ Validation      │   │ Fast projections   │
                    │ Transactions    │   │ Eventual sync      │
                    └────────┬────────┘   └──────────────────┘
                             │                    ▲
                             │    events/sync      │
                             └────────────────────┘
```

---

## Commands vs Queries — The Fundamental Split

```
Commands:                           Queries:
────────────────────────────────────────────────────────────────
PlaceOrderCommand                   GetOrderByIdQuery
CancelOrderCommand                  GetOrdersByUserQuery
ProcessPaymentCommand               GetOrdersDashboardQuery
UpdateProductPriceCommand           SearchOrdersByStatusQuery
ReserveStockCommand                 GetRevenueReportQuery

Characteristics:                    Characteristics:
→ Mutates state                     → Never mutates state
→ Returns void or ID only           → Returns data
→ Can fail (validation, business    → Can be retried freely
  rules)                            → Can be cached
→ Goes through domain model         → Bypasses domain model
→ Validated strictly                → Shaped for consumer
```

**The query side never goes through the domain model** — it reads directly from a projection optimized for that exact query. No domain object instantiation. No business rule checking. Just fast data retrieval.

---

## CQRS Without Event Sourcing — The Simple Form

CQRS doesn't require event sourcing. The simplest form just uses **two separate data access patterns** against the same database:

```
Same PostgreSQL DB, but:
  Writes → go through domain model, JPA, validation
  Reads  → go through raw JPQL/SQL projections, DTOs
```

```java
// WRITE SIDE — rich domain model, full validation
@Service
public class OrderCommandService {

    @Autowired OrderRepository orderRepository; // JPA entity-based

    @Transactional
    public UUID placeOrder(PlaceOrderCommand cmd) {
        // validate through domain model
        Product product = productRepository.findById(cmd.getProductId())
            .orElseThrow(() -> new ProductNotFoundException(...));

        if (product.getStock() < cmd.getQuantity()) {
            throw new InsufficientStockException(...);
        }

        Order order = Order.create(cmd.getUserId(), product, cmd.getQuantity());
        orderRepository.save(order);
        return order.getId();
    }
}
```

```java
// READ SIDE — lean DTO projection, no domain model
@Service
public class OrderQueryService {

    @Autowired EntityManager em;

    // Returns exactly the shape the UI needs — no mapping overhead
    public OrderSummaryDto getOrderSummary(UUID orderId) {
        return em.createQuery("""
            SELECT new com.shopsphere.dto.OrderSummaryDto(
                o.id, o.status, o.amount,
                u.name, u.email,
                p.name, p.imageUrl
            )
            FROM Order o
            JOIN o.user u
            JOIN o.product p
            WHERE o.id = :orderId
            """, OrderSummaryDto.class)
            .setParameter("orderId", orderId)
            .getSingleResult();
    }

    // Admin dashboard — complex aggregation, bypasses domain entirely
    public List<OrderDashboardRow> getDashboard(LocalDate from, LocalDate to) {
        return em.createNativeQuery("""
            SELECT
                DATE(o.created_at)   AS order_date,
                COUNT(*)             AS total_orders,
                SUM(o.amount)        AS total_revenue,
                AVG(o.amount)        AS avg_order_value,
                COUNT(CASE WHEN o.status = 'CANCELLED' THEN 1 END) AS cancellations
            FROM orders o
            WHERE o.created_at BETWEEN :from AND :to
            GROUP BY DATE(o.created_at)
            ORDER BY order_date DESC
            """, OrderDashboardRow.class)
            .setParameter("from", from)
            .setParameter("to", to)
            .getResultList();
    }
}
```

Even without a separate read database, this already gives you:
- Clean separation of concerns
- Read queries optimized independently of domain model
- Domain model complexity doesn't bleed into read paths
- Read side easily cacheable

---

## CQRS With Separate Read Store — The Full Form

For high-read services, the read model moves to a **completely separate store** optimized for reads:

```
Write side:  PostgreSQL (normalized, transactional)
Read side:   Elasticsearch / Redis / MongoDB / Read replica

Sync mechanism: events published on write → projection updates read store
```

```
ShopSphere order-service — full CQRS:

┌──────────────────────────────────────────────────────────────┐
│                      order-service                            │
│                                                               │
│  PlaceOrderCommand ──▶ OrderCommandService                   │
│                              │                                │
│                              ▼                                │
│                        PostgreSQL ──publishes──▶ Kafka        │
│                        (write DB)    OrderCreated             │
│                                                               │
│  GetOrderQuery ──▶ OrderQueryService                         │
│                          │                                    │
│                          ▼                                    │
│                     Redis cache                               │
│                    (read store)                               │
└──────────────────────────────────────────────────────────────┘

Kafka OrderCreated event
      │
      ▼
 OrderProjectionHandler
      │
      ├──▶ updates Redis (order detail cache)
      ├──▶ updates Elasticsearch (order search index)
      └──▶ updates analytics DB (revenue projections)
```

---

## Projection Handler — Keeping Read Side In Sync

```java
// Listens to write-side events, updates read models
@Component
@Slf4j
public class OrderProjectionHandler {

    @Autowired RedisTemplate<String, OrderReadDto> redis;
    @Autowired OrderSearchRepository elasticsearchRepo;

    @KafkaListener(topics = "order-events")
    public void handle(OrderEvent event) {
        if (event instanceof OrderCreatedEvent e) {
            onOrderCreated(e);
        } else if (event instanceof OrderStatusChangedEvent e) {
            onStatusChanged(e);
        } else if (event instanceof OrderCancelledEvent e) {
            onOrderCancelled(e);
        }
    }

    private void onOrderCreated(OrderCreatedEvent e) {
        OrderReadDto dto = OrderReadDto.builder()
            .orderId(e.getOrderId())
            .userId(e.getUserId())
            .productName(e.getProductName())
            .amount(e.getAmount())
            .status("PLACED")
            .createdAt(e.getOccurredAt())
            .build();

        // update Redis — fast single-order lookups
        redis.opsForValue().set(
            "order:" + e.getOrderId(),
            dto,
            Duration.ofHours(24)
        );

        // update Elasticsearch — searchable order index
        elasticsearchRepo.save(toSearchDoc(dto));
    }

    private void onStatusChanged(OrderStatusChangedEvent e) {
        // partial update — don't reload full order, just patch status
        String key = "order:" + e.getOrderId();
        OrderReadDto dto = redis.opsForValue().get(key);
        if (dto != null) {
            dto.setStatus(e.getNewStatus());
            redis.opsForValue().set(key, dto, Duration.ofHours(24));
        }
        elasticsearchRepo.updateStatus(e.getOrderId(), e.getNewStatus());
    }
}
```

```java
// Read side — query service hits read store, never PostgreSQL
@Service
public class OrderQueryService {

    @Autowired RedisTemplate<String, OrderReadDto> redis;
    @Autowired OrderSearchRepository elasticsearchRepo;
    @Autowired OrderJpaRepository postgresRepo; // fallback only

    public OrderReadDto getOrder(UUID orderId) {
        // 1. Try Redis first (fastest)
        String key = "order:" + orderId;
        OrderReadDto cached = redis.opsForValue().get(key);
        if (cached != null) return cached;

        // 2. Cache miss → try Elasticsearch
        Optional<OrderSearchDoc> doc = elasticsearchRepo.findById(orderId);
        if (doc.isPresent()) return toDto(doc.get());

        // 3. Last resort — PostgreSQL (source of truth)
        Order order = postgresRepo.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        OrderReadDto dto = toDto(order);
        redis.opsForValue().set(key, dto, Duration.ofHours(24)); // repopulate cache
        return dto;
    }

    public Page<OrderSearchDoc> searchOrders(OrderSearchRequest req) {
        // Elasticsearch handles complex search — PostgreSQL never touched
        return elasticsearchRepo.search(
            req.getUserId(),
            req.getStatus(),
            req.getFromDate(),
            req.getToDate(),
            req.getPageable()
        );
    }
}
```

---

## Read Model Shapes — Different Views Per Consumer

The same domain data needs different shapes for different consumers. CQRS makes this natural:

```
Write model: Order entity (normalized, domain-complete)

Read models:
──────────────────────────────────────────────────────────────────
OrderDetailDto        ← full details for order detail page
  orderId, status, amount, product details, user info,
  payment info, shipping address, timeline of events

OrderSummaryDto       ← compact for order list page
  orderId, status, amount, productName, createdAt

OrderDashboardRow     ← aggregated for admin analytics
  date, totalOrders, totalRevenue, cancellationRate

OrderMobileDto        ← minimal for mobile app
  orderId, status, shortDescription, amount

ShippingQueueItem     ← shaped for fulfillment team
  orderId, shippingAddress, productSku, priority
──────────────────────────────────────────────────────────────────

Each is a separate projection, independently maintained.
Adding a new consumer view = add a new projection handler.
Zero impact on write side.
```

---

## Eventual Consistency on the Read Side

The read model is **eventually consistent** with the write model. The sync happens asynchronously via events.

```
Timeline:
─────────────────────────────────────────────────────────────────
T=0:    Client sends PlaceOrderCommand
T=1ms:  PostgreSQL write committed → order exists in write DB
T=2ms:  OrderCreated event published to Kafka
T=1ms:  Command handler returns orderId to client ✅
                   ↓
T=50ms: Kafka delivers event to projection handler
T=51ms: Redis updated with new order
T=52ms: Elasticsearch indexed

T=10ms: Client immediately queries GET /orders/{id}
        → Redis not yet updated → cache miss
        → Falls back to PostgreSQL ← still consistent ✅

T=100ms: All subsequent reads → Redis hit ✅
─────────────────────────────────────────────────────────────────
```

For commands that must immediately return readable state (like "show me my order after placing it") — fall back to the write DB for that specific read, or include data in the command response directly.

---

## CQRS Folder Structure in ShopSphere

```
order-service/
  src/main/java/com/shopsphere/order/
  │
  ├── command/                         ← WRITE SIDE
  │   ├── PlaceOrderCommand.java
  │   ├── CancelOrderCommand.java
  │   ├── OrderCommandService.java     ← handles commands
  │   └── OrderCommandController.java  ← POST /orders
  │
  ├── query/                           ← READ SIDE
  │   ├── GetOrderQuery.java
  │   ├── OrderSummaryDto.java
  │   ├── OrderDetailDto.java
  │   ├── OrderQueryService.java       ← handles queries
  │   └── OrderQueryController.java    ← GET /orders/{id}
  │
  ├── domain/                          ← DOMAIN MODEL (write side only)
  │   ├── Order.java                   ← rich entity
  │   ├── OrderStatus.java
  │   └── OrderRepository.java         ← JPA
  │
  ├── projection/                      ← READ SIDE SYNC
  │   ├── OrderProjectionHandler.java  ← Kafka listener
  │   ├── OrderReadDto.java            ← Redis-cached shape
  │   └── OrderSearchDocument.java     ← Elasticsearch shape
  │
  └── event/                           ← DOMAIN EVENTS
      ├── OrderCreatedEvent.java
      ├── OrderStatusChangedEvent.java
      └── OrderCancelledEvent.java
```

---

## CQRS — When to Use It

```
✅ Use CQRS when:

  High read/write ratio (reads >> writes)
    → Optimize read path independently
    → ShopSphere: 1 order write → 100 reads of order status

  Complex read requirements
    → Dashboard aggregations, multi-join reports
    → Different shapes for different consumers

  Read and write scale differently
    → Scale read replicas independently of write primary
    → Add Redis/Elasticsearch without touching write path

  Event Sourcing is already in use
    → Projections are the natural read model for ES

  Domain model is complex
    → Don't pollute domain with read concerns
    → Keep domain focused on invariants and business rules

❌ Don't use CQRS when:

  Simple CRUD with no complex reads
    → Overhead not worth it for basic admin panels
    → user-service profile management — just CRUD

  Small team, early stage product
    → Premature optimization
    → Add CQRS when read/write problems actually appear

  Strong consistency required for reads
    → If you cannot tolerate any staleness on reads
    → CQRS introduces eventual consistency on read side
```

---

## CQRS + ES + Saga — The Full Picture

These three patterns work together as a system:

```
User: "Place an order"
        │
        ▼
PlaceOrderCommand
        │
        ▼
OrderCommandService (WRITE SIDE)
  → validates via domain model
  → appends OrderPlacedEvent to event store  ← EVENT SOURCING
  → publishes event to Kafka                 ← OUTBOX PATTERN
        │
        ▼
Kafka: OrderCreatedEvent
  │
  ├──▶ OrderSagaOrchestrator         ← SAGA (coordinates distributed tx)
  │       → ReserveStockCommand
  │       → ProcessPaymentCommand
  │       → CompensateIfNeeded
  │
  └──▶ OrderProjectionHandler        ← CQRS READ SIDE SYNC
          → updates Redis
          → updates Elasticsearch

User: "What's my order status?"
        │
        ▼
GetOrderQuery
        │
        ▼
OrderQueryService (READ SIDE)
  → hits Redis (sub-millisecond)
  → returns OrderReadDto ✅
```

Each pattern solves a distinct problem. Together they form a production-grade distributed order system.

---

## Interview Angles 🎯

**Q: What is CQRS and what problem does it solve?**
> CQRS separates the model used for writes (commands) from the model used for reads (queries). A single model optimized for transactional writes — normalized, domain-rich, validated — is inherently slow for complex reads. CQRS lets you optimize each side independently: a normalized write model for consistency and integrity, and denormalized projections for fast, flexible reads.

**Q: Does CQRS require Event Sourcing?**
> No — they're independent patterns that complement each other. The simplest CQRS just uses two different access patterns against the same database: a rich domain model for writes and raw SQL/JPQL projections for reads. Full CQRS with a separate read store and async sync via events is more powerful but also more complex. Start simple, add the separate store when you have a concrete read performance problem.

**Q: What's the consistency model of the CQRS read side?**
> Eventual consistency. The read model is updated asynchronously when events are published from the write side. There's a window — typically milliseconds to seconds — where the read side lags behind. For most use cases this is acceptable. For cases where it isn't (immediately after a write), fall back to the write database for that specific read or return needed data directly in the command response.

**Q: How does CQRS help in a high-read system like ShopSphere?**
> Order placement is infrequent relative to order status checks. With CQRS, the write path goes through a fully validated domain model into PostgreSQL. The read path hits Redis (sub-millisecond) served from a projection built from Kafka events. The read infrastructure scales independently — more Redis replicas, more projection consumers — without touching write-side complexity or performance.

**Q: How do you handle a query that needs data immediately after a command, before the projection updates?**
> Three options: return the necessary data in the command response itself (return order ID + status from PlaceOrder); include a version or sequence number in the command response and poll until the read model catches up to that version; or fall back to the write database for that specific post-command read. The first option is the simplest and most common.

---

Say **next** for **Topic 12: Outbox Pattern** — reliably publishing events even if your service crashes mid-write 🚀
