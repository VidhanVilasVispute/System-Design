## Topic 5 — Data Ownership

### The Problem First

In a monolith, one big shared database is normal:

```
MONOLITH DB
┌─────────────────────────────────────────┐
│  users table                            │
│  orders table  ← Order code JOINs here  │
│  products table                         │
│  payments table                         │
│  reviews table                          │
└─────────────────────────────────────────┘

Any module can read/write any table.
JOINs are free. Transactions are trivial.
```

When teams move to microservices but keep the shared DB:

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Order        │   │ Product      │   │ User         │
│ Service      │   │ Service      │   │ Service      │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                   │
       └──────────────────┼───────────────────┘
                          │
                 ┌────────▼────────┐
                 │  SHARED DB      │  ← The anti-pattern
                 └─────────────────┘
```

This is the **Shared Database anti-pattern**. It looks convenient but destroys microservice independence entirely.

---

### Why Shared Database Breaks Microservices

#### 1. Deployment Coupling
```
Product team needs to rename column: product_name → title

But Order Service also SELECTs product_name directly.
→ You cannot deploy Product Service change without coordinating
  with Order Service team.
→ You've lost independent deployment. Congratulations, you have
  a distributed monolith — worst of both worlds.
```

#### 2. Schema Coupling
```
Order Service does:
  SELECT u.email FROM users u
  JOIN orders o ON o.user_id = u.id
  WHERE o.id = ?

User Service now CANNOT change the users table schema
without breaking Order Service's query.
Every service is now a hidden dependency of every other service.
```

#### 3. No Independent Scaling
```
Search queries hammer the shared DB.
Order writes also hammer the same shared DB.
You cannot tune the DB for one workload without affecting the other.
You cannot use Elasticsearch for search while using PostgreSQL for orders
  — everything is forced into one technology.
```

---

### The Rule: Database Per Service

Each service **owns its data exclusively**. No other service touches it directly.

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Order        │    │ Product      │    │ User         │
│ Service      │    │ Service      │    │ Service      │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                    │
       ▼                   ▼                    ▼
┌────────────┐    ┌────────────────┐    ┌────────────┐
│ Orders DB  │    │  Products DB   │    │  Users DB  │
│ PostgreSQL │    │  PostgreSQL    │    │ PostgreSQL │
│            │    │  +Elasticsearch│    │            │
└────────────┘    └────────────────┘    └────────────┘

Rules:
  Order Service → can ONLY access Orders DB
  Product Service → can ONLY access Products DB
  If Order Service needs product info → it calls Product Service API
```

---

### What "Ownership" Actually Means

```
The owning service is the SINGLE SOURCE OF TRUTH for its data.

Product Service owns:
  - The products table schema
  - All write operations (create, update, delete products)
  - Read access via its own API
  - Migration decisions (can rename columns freely)
  - Choice of storage technology

No one else:
  - Writes to products table directly
  - Reads products table via SQL directly
  - Has credentials to the products DB
```

---

### Polyglot Persistence

Since each service owns its DB, each service can choose the **right storage technology** for its use case:

```
┌──────────────────┬─────────────────────────────────────────┐
│ Service          │ Storage Choice + Reason                  │
├──────────────────┼─────────────────────────────────────────┤
│ user-service     │ PostgreSQL — relational, ACID, auth data │
│ order-service    │ PostgreSQL — transactions, consistency   │
│ product-service  │ PostgreSQL + Elasticsearch — search      │
│ session/cache    │ Redis — low latency key-value            │
│ notification     │ No DB — stateless, RabbitMQ consumer     │
│ review-service   │ PostgreSQL — structured, queryable       │
│ analytics        │ Cassandra/ClickHouse — time-series, OLAP │
│ file storage     │ S3 — blob storage                        │
└──────────────────┴─────────────────────────────────────────┘
```

This is **Polyglot Persistence** — many storage technologies, each chosen for the workload it serves.

---

### The Data Duplication Reality

When services can't share tables, they sometimes need **copies of each other's data**:

```
Order Service needs to display:
  - orderId, quantity, price        ← owned by Order Service ✅
  - productName, productImage       ← owned by Product Service ❌

Option 1: Call Product Service API every time (API Composition)
  Pro: Always fresh data
  Con: Latency, coupling at runtime, Product Service down = Order broken

Option 2: Order Service stores a COPY of product name + image
  when the order is placed

  orders table:
    order_id | product_id | product_name_snapshot | quantity | price
      1001   |   prod-55  | "Nike Air Max"        |    2     | 8999

  Pro: Order Service fully autonomous, no runtime dependency
  Con: If product name changes later, Order has stale copy
       → BUT this is correct for orders — you want the name
         AT THE TIME OF PURCHASE, not the current name
```

**Data duplication in microservices is not always a bug — it's often intentional and correct.**

---

### Keeping Copies in Sync — Events

When Product Service updates a product, how does other services' copies stay updated?

```
Product Service updates product name:
  1. Updates its own DB
  2. Publishes event: ProductUpdatedEvent { productId, newName, newPrice }
         │
         ▼
     Kafka Topic: product-events
         │
         ├──▶ Search Service consumes → updates Elasticsearch index
         ├──▶ Order Service consumes → updates product_name_snapshot
         └──▶ Cart Service consumes  → updates cart item display name

Each service updates its own copy asynchronously.
This is EVENTUAL CONSISTENCY — copies will lag behind
  by milliseconds to seconds, but will converge.
```

---

### Eventual Consistency Trade-off

```
Strong Consistency (shared DB):
  All services see the same data at the same instant.
  Achieved via shared DB or distributed transactions.
  Cost: Tight coupling, can't scale independently, tech lock-in.

Eventual Consistency (events + data duplication):
  All services will have the same data — eventually.
  There's a window (ms to seconds) where copies diverge.
  Cost: Your system must tolerate brief inconsistency.

For most business operations, eventual consistency is fine:
  Product name update propagates in 200ms → nobody notices
  
For some operations, it's not acceptable:
  Bank balance must be immediately consistent across all reads
  → Those cases need special handling (Saga, 2PC, or keeping
    in ONE service's DB instead of splitting)
```

---

### The Saga Pattern (Brief Preview)

What if a business operation spans multiple services?

```
Place Order flow:
  1. Create order record     → Order Service DB
  2. Reserve inventory       → Inventory Service DB
  3. Charge payment          → Payment Service DB
  4. Schedule delivery       → Delivery Service DB

All four must succeed or ALL must be rolled back.
No shared DB transaction possible.
```

**Saga** breaks this into a sequence of local transactions with **compensating transactions** for rollback:

```
Step 1: Order created         ✅  (compensate: cancel order)
Step 2: Inventory reserved    ✅  (compensate: release inventory)
Step 3: Payment charged       ✅  (compensate: refund payment)
Step 4: Delivery scheduled    ❌  FAILS

Saga rolls back in reverse:
  → Refund payment (compensate step 3)
  → Release inventory (compensate step 2)
  → Cancel order (compensate step 1)
```

Full Saga deep dive is its own topic (Stage 9 territory) — just know it exists as the solution to cross-service transactions.

---

### Boundaries — How to Decide What Goes Where

A useful heuristic: **who is the natural owner of this data?**

```
Which service should own "order status"?
  → Order Service. It's the one that changes it.
     (created → paid → shipped → delivered)

Which service should own "product price"?
  → Product Service. Pricing is a product concern.

Which service should own "user's order history"?
  → Order Service stores the orders.
     User Service stores the user profile.
     Order history = query Order Service by userId.
     NOT stored in User Service.

Rule: Data lives with the service that is responsible
      for its lifecycle (create, update, delete).
```

---

### ShopSphere Lens

```
ShopSphere already follows this correctly:

user-service     → owns users table         → nobody else writes to it
product-service  → owns products table      → search-service has its OWN
                                               Elasticsearch index (copy)
order-service    → owns orders table        → stores product_id reference,
                                               not a foreign key to products DB
review-service   → owns reviews table       → stores user_id + product_id
                                               as plain columns, not FK joins

When product-service updates a product:
  → Kafka event → search-service updates Elasticsearch
  → This is exactly the event-driven data sync pattern

Your FeignAuthInterceptor calls APIs — never direct DB access.
This is the pattern working correctly.
```

---

### Interview Questions

**Q1. Why should each microservice have its own database?**

> Shared databases create schema coupling, deployment coupling, and prevent independent scaling. If Service A can directly read/write Service B's tables, they're not truly independent — a schema change in one breaks the other. Database-per-service enforces true ownership boundaries and lets each service choose the right storage technology.

**Q2. Isn't data duplication bad? Microservices seem to encourage it.**

> In a relational monolith, duplication is a bug — it causes update anomalies. In microservices, controlled duplication via event-driven sync is intentional. Each service keeps a local copy of the data it needs, updated asynchronously via events. The trade-off is eventual consistency instead of strong consistency, which is acceptable for most business data.

**Q3. How do you handle a transaction that spans multiple microservices?**

> You can't use a single DB transaction across service boundaries. The Saga pattern handles this — each service performs a local transaction and publishes an event. If a later step fails, compensating transactions roll back previous steps in reverse order. It's more complex than a DB transaction but maintains data consistency without shared state.

**Q4. How do you decide where a piece of data belongs?**

> Data belongs to the service responsible for its full lifecycle — the one that creates, updates, and deletes it. Other services that need that data either call the owning service's API at runtime or maintain an eventually consistent local copy updated via domain events.

---

Ready for **Topic 6 — Distributed Tracing** when you are.
