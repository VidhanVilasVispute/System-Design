# Stage 3 — Topic 7: SQL vs NoSQL

## Theory

SQL vs NoSQL is one of the most asked system design questions — and one of the most poorly answered. Most candidates treat it as a binary choice with a simple rule like "SQL for structured data, NoSQL for scale." That is surface-level. The real answer requires understanding **why** each model makes the trade-offs it does, what those trade-offs cost in production, and how to choose intelligently based on access patterns.

**The core philosophical difference:**

```
SQL (Relational):
  Data is organised into tables with fixed schemas.
  Relationships between entities are explicit and enforced.
  The database guarantees correctness — ACID transactions,
  foreign key constraints, referential integrity.
  You define structure upfront. The DB enforces it forever.

NoSQL (Non-Relational):
  Data is organised around access patterns, not relationships.
  Schema is flexible — or absent entirely.
  The application is responsible for correctness.
  You optimise for how data is read, not how it relates.
  Scale and performance are first-class concerns.
```

Neither is universally better. They solve different problems. The engineer's job is to match the database to the problem.

---

## SQL — Relational Databases Deep Dive

### How Relational Databases Work

A relational database organises data into **tables** (relations) with rows and columns. Every row has a fixed set of columns defined by the schema.

```
orders table:
┌──────────┬─────────┬───────────┬─────────────────────┬──────────┐
│ id       │ user_id │ status    │ created_at          │ total    │
├──────────┼─────────┼───────────┼─────────────────────┼──────────┤
│ o-789    │ u-123   │ CONFIRMED │ 2026-04-08 10:30:00 │ 1299.99  │
│ o-790    │ u-456   │ PENDING   │ 2026-04-08 10:31:00 │ 549.50   │
│ o-791    │ u-123   │ SHIPPED   │ 2026-04-08 10:32:00 │ 899.00   │
└──────────┴─────────┴───────────┴─────────────────────┴──────────┘

order_items table:
┌────────┬──────────┬────────────┬──────────┬──────────┐
│ id     │ order_id │ product_id │ quantity │ price    │
├────────┼──────────┼────────────┼──────────┼──────────┤
│ oi-001 │ o-789    │ p-456      │ 2        │ 649.99   │
│ oi-002 │ o-790    │ p-789      │ 1        │ 549.50   │
│ oi-003 │ o-791    │ p-123      │ 1        │ 899.00   │
└────────┴──────────┴────────────┴──────────┴──────────┘
```

**The power of relations — JOIN:**
```sql
-- Get all orders with customer names and product names
SELECT
    o.id          AS order_id,
    u.name        AS customer_name,
    p.name        AS product_name,
    oi.quantity,
    oi.price,
    o.status
FROM orders o
JOIN users u        ON o.user_id    = u.id
JOIN order_items oi ON o.id         = oi.order_id
JOIN products p     ON oi.product_id = p.id
WHERE o.status = 'CONFIRMED'
ORDER BY o.created_at DESC;

-- One query assembles data from 4 tables
-- The DB handles the join — no application code needed
-- Referential integrity guarantees every foreign key exists
```

### ACID — The Relational Guarantee

ACID is what you are buying when you choose a relational database:

**Atomicity — all or nothing:**
```sql
-- Transfer money between accounts
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 'acc-001';
    UPDATE accounts SET balance = balance + 500 WHERE id = 'acc-002';
COMMIT;

-- If the second UPDATE fails, the first is rolled back
-- Money never disappears from one account without appearing in another
-- The transaction is atomic — it either completes entirely or not at all
```

**Consistency — database rules are never violated:**
```sql
-- Foreign key constraint
ALTER TABLE order_items
ADD CONSTRAINT fk_order
FOREIGN KEY (order_id) REFERENCES orders(id);

-- Attempt to insert an order item for non-existent order:
INSERT INTO order_items (order_id, product_id, quantity)
VALUES ('non-existent-order', 'p-456', 1);
-- ERROR: insert or update on table "order_items" violates
--        foreign key constraint "fk_order"

-- Consistency: database cannot be left in an invalid state
-- The application cannot accidentally create orphaned records
```

**Isolation — concurrent transactions don't see each other's intermediate state:**
```sql
-- Two concurrent transactions:
-- T1: deduct inventory
-- T2: check inventory

-- Without isolation (dirty read problem):
T2 reads: inventory = 10 (T1 has deducted but not committed)
T1 rolls back: inventory = 10 again
T2 thinks inventory = 9 (wrong — saw uncommitted data)

-- With READ COMMITTED isolation:
T2 only sees committed data — no dirty reads
T1's intermediate state is invisible to T2
```

**Isolation levels:**
```
READ UNCOMMITTED:  Can read uncommitted changes — dirty reads possible
READ COMMITTED:    Only reads committed data — default in PostgreSQL
REPEATABLE READ:   Same row reads return same data within transaction
SERIALIZABLE:      Full isolation — transactions appear sequential

Higher isolation = more consistency = lower concurrency
Lower isolation  = more performance = potential anomalies

PostgreSQL default: READ COMMITTED
MySQL InnoDB default: REPEATABLE READ
```

**Durability — committed data survives crashes:**
```
When COMMIT returns:
  Data is written to the Write-Ahead Log (WAL) on disk
  Even if the server crashes immediately after COMMIT
  The transaction will be recovered on restart from WAL

PostgreSQL WAL flow:
  1. Transaction modifies pages in memory (buffer pool)
  2. WAL record written to disk — BEFORE modifying actual pages
  3. COMMIT returns to application
  4. Pages written to actual data files eventually (checkpoint)
  
  If crash between steps 3 and 4:
    WAL replayed on restart → pages reconstructed → no data loss
```

### When SQL Is the Right Choice

```
Use a relational database when:

1. Data has complex relationships:
   Orders → OrderItems → Products → Categories
   Users → Roles → Permissions
   Complex JOIN queries are natural and efficient

2. Transactions span multiple entities:
   Debit account A, credit account B — must be atomic
   Place order + decrement inventory — must be consistent

3. Schema is well-understood and stable:
   Financial records, order management, user accounts
   Data structure known upfront, changes infrequently

4. Ad-hoc queries matter:
   Analytics, reporting, admin dashboards
   SQL lets you ask any question without schema redesign

5. Data integrity is non-negotiable:
   Financial systems, healthcare, legal records
   Wrong data costs money or lives
   
ShopSphere uses PostgreSQL for:
  User accounts, orders, order items, payments, products
  All entities with complex relationships and transaction requirements
```

---

## NoSQL — Four Categories Deep Dive

NoSQL is not one thing — it is four fundamentally different data models, each optimised for a specific problem.

### Category 1 — Key-Value Stores

**Model:** Dictionary / HashMap. Every value stored and retrieved by a unique key.

```
key                        value
─────────────────────────────────────────────────
"session:u-123"          → { userId, roles, cart, expiresAt }
"rate_limit:u-456"       → 47 (request count)
"product:p-789:cache"    → { full product object JSON }
"lock:order:o-999"       → "processing" (distributed lock)
```

**Characteristics:**
```
Extremely fast: O(1) lookup by key
No query language: cannot search by value, no JOINs
No schema: value is a blob — JSON, binary, anything
Horizontal scaling: trivially shard by key hash
Best for: caching, sessions, rate limiting, feature flags

Redis capabilities beyond pure key-value:
  Strings, Hashes, Lists, Sets, Sorted Sets, Streams
  TTL per key
  Pub/Sub
  Lua scripting for atomic operations
  Persistence (RDB + AOF)
```

**ShopSphere uses Redis for:**
```java
// Session storage
redisTemplate.opsForValue().set(
    "session:" + sessionId, session, Duration.ofHours(24));

// Rate limiting counter
Long count = redisTemplate.opsForValue()
    .increment("rate_limit:" + userId);

// Distributed lock
Boolean locked = redisTemplate.opsForValue()
    .setIfAbsent("lock:order:" + orderId, "1", Duration.ofSeconds(30));

// Shopping cart (Hash — field per item)
redisTemplate.opsForHash().put(
    "cart:" + userId, productId, quantity);

// Sorted set — leaderboard by sales
redisTemplate.opsForZSet().incrementScore(
    "seller:leaderboard", sellerId, orderTotal);
```

---

### Category 2 — Document Stores

**Model:** Collections of self-contained documents (JSON/BSON). Each document can have a different structure.

```javascript
// MongoDB document — order with all related data embedded
{
  "_id": "o-789",
  "userId": "u-123",
  "status": "CONFIRMED",
  "total": 1299.99,
  "createdAt": "2026-04-08T10:30:00Z",

  // Embedded array — no JOIN needed
  "items": [
    {
      "productId": "p-456",
      "name": "Nike Air Max",
      "price": 649.99,
      "quantity": 2,
      "thumbnail": "https://..."
    }
  ],

  // Embedded object
  "shippingAddress": {
    "line1": "123 Main St",
    "city": "Nagpur",
    "state": "Maharashtra",
    "pin": "440001"
  },

  // Flexible schema — can add new fields without migration
  "promoCode": "SAVE10",
  "loyaltyPointsEarned": 1300
}
```

**Why embedding beats JOINs for read-heavy workloads:**
```
Relational:
  Fetch order with items:
    SELECT * FROM orders WHERE id = 'o-789'           -- query 1
    SELECT * FROM order_items WHERE order_id = 'o-789' -- query 2
    SELECT * FROM products WHERE id IN (p-456, ...)    -- query 3
    3 queries, 2 JOINs, latency compounds

Document:
  Fetch order with items:
    db.orders.findOne({ _id: "o-789" })
    1 query, 0 JOINs, all data in one read
    
Faster reads at the cost of write complexity and data duplication.
```

**Flexible schema — migrations without downtime:**
```javascript
// Old documents have field "address"
{ "_id": "o-100", "address": "123 Main St" }

// New documents have structured "shippingAddress"
{ "_id": "o-200", "shippingAddress": { "line1": "...", "city": "..." } }

// Both coexist in same collection — no migration required
// Application handles both shapes:
const address = order.shippingAddress?.line1 || order.address;
```

**MongoDB query capabilities:**
```javascript
// Rich query language — not just key lookup
db.orders.find({
    status: "CONFIRMED",
    total: { $gt: 1000 },
    "items.productId": "p-456",
    createdAt: { $gte: new Date("2026-04-01") }
})
.sort({ createdAt: -1 })
.limit(20)

// Aggregation pipeline — analytics within MongoDB
db.orders.aggregate([
    { $match: { status: "CONFIRMED" } },
    { $group: {
        _id: "$userId",
        totalSpent: { $sum: "$total" },
        orderCount: { $sum: 1 }
    }},
    { $sort: { totalSpent: -1 } },
    { $limit: 10 }
])
// Top 10 customers by spend — no SQL needed
```

**When to use document stores:**
```
Content management systems — articles, blog posts, product catalogs
  Each document has different fields — flexible schema needed
  
User profiles — different users have different attributes
  One user has a business profile, another has individual profile
  
Product catalogs — electronics have different specs than clothing
  No fixed schema — each product type has unique fields
  
Event logs — each event type has different payload shape
  
ShopSphere uses MongoDB for:
  Product catalog (different product types have different attributes)
  User activity logs (flexible event shapes)
  CMS content (pages, banners, promotions — all different shapes)
```

---

### Category 3 — Column-Family Stores

**Model:** Data stored by column families rather than rows. Optimised for write-heavy workloads and time-series data.

**Cassandra architecture:**
```
Traditional row storage (PostgreSQL):
  Row 1: [id=1, name="Vidhan", email="v@e.com", age=25]
  Row 2: [id=2, name="Alice",  email="a@e.com", age=30]
  Stored row by row on disk

Column-family storage (Cassandra):
  Partition key determines physical location on ring
  Within a partition, data sorted by clustering key

  Partition: user_id = "u-123"
    ┌───────────────────────────────────────────────────┐
    │ 2026-04-01: { event: LOGIN,  ip: 1.2.3.4 }        │
    │ 2026-04-02: { event: ORDER,  orderId: o-789 }     │
    │ 2026-04-03: { event: SEARCH, query: "shoes" }     │
    └───────────────────────────────────────────────────┘
    Rows within partition sorted by timestamp
    Extremely fast range scans within a partition
```

**Cassandra's CQL — SQL-like but very different semantics:**
```cql
-- Table design in Cassandra is ACCESS PATTERN FIRST
-- "What queries will we run?" determines table design

-- Get all orders for a user (access pattern: by userId)
CREATE TABLE orders_by_user (
    user_id     TEXT,
    created_at  TIMESTAMP,
    order_id    TEXT,
    status      TEXT,
    total       DECIMAL,
    PRIMARY KEY (user_id, created_at)  -- user_id=partition key, created_at=sort key
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Query perfectly matches table design:
SELECT * FROM orders_by_user
WHERE user_id = 'u-123'          -- hits exactly one partition
  AND created_at >= '2026-04-01' -- range scan within partition
LIMIT 20;

-- This is FAST: single partition lookup + sequential scan
-- The SAME query in Cassandra is very different from SQL:
-- Cannot add: AND status = 'CONFIRMED' without a different table
-- No JOINs — ever
-- No aggregations across partitions
```

**Cassandra's strengths and limitations:**
```
Strengths:
  Writes: extremely fast — append-only LSM tree structure
  Horizontal scale: linear — add nodes, throughput increases proportionally
  No single point of failure — every node is equal (no master)
  Geographic distribution — multi-datacenter replication built-in
  Time-series data: perfect fit — ordered by time within partition

Limitations:
  No JOINs — denormalization required
  No transactions across partitions
  Query flexibility severely limited by data model
  Cannot query by non-partition-key columns without secondary index
    (secondary indexes are expensive — avoid in production)
  Schema design is hard — must know all access patterns upfront
  
Best for:
  IoT sensor data, time-series metrics
  User activity logs, audit trails
  Message history (WhatsApp stores messages in Cassandra)
  Any write-heavy, time-ordered data at massive scale
```

---

### Category 4 — Graph Databases

**Model:** Nodes (entities) and edges (relationships). Query by traversing relationships.

```
Graph model in Neo4j:

Nodes:
  (u:User {id: "u-123", name: "Vidhan"})
  (p:Product {id: "p-456", name: "Nike Shoes"})
  (c:Category {name: "Footwear"})
  (s:Seller {id: "s-789", name: "Nike Store"})

Edges (Relationships):
  (u)-[:PURCHASED {at: "2026-04-08"}]->(p)
  (u)-[:VIEWED {count: 5}]->(p)
  (p)-[:BELONGS_TO]->(c)
  (p)-[:SOLD_BY]->(s)
  (u)-[:FOLLOWS]->(s)

Cypher query — "What did users who bought the same shoes as Vidhan also buy?":
MATCH (u:User {id: "u-123"})-[:PURCHASED]->(p:Product)<-[:PURCHASED]-(other:User)
      -[:PURCHASED]->(rec:Product)
WHERE NOT (u)-[:PURCHASED]->(rec)
RETURN rec.name, COUNT(other) AS purchaseCount
ORDER BY purchaseCount DESC
LIMIT 10

This is the recommendation engine query.
In SQL: 3 self-joins on a purchase table — O(n³)
In Neo4j: graph traversal — O(depth × branching_factor)
Dramatically faster for deep relationship queries
```

**When graph databases shine:**
```
Social networks: followers, following, mutual connections
  "Find friends of friends who live in Mumbai"
  
Recommendation engines:
  "Users who bought X also bought Y" — collaborative filtering
  
Fraud detection:
  "Find all accounts connected to this suspicious account
   within 3 degrees of separation" — graph traversal
   
Knowledge graphs, semantic search
Access control with complex inheritance
Network topology (infrastructure, dependencies)

ShopSphere uses Neo4j for:
  Recommendation engine — "customers also bought"
  Seller relationship graph — brand affiliations
  Fraud detection — suspicious account clusters
```

---

## The CAP Theorem Preview

Different databases make different trade-offs that directly relate to CAP:

```
CAP Theorem (preview — covered fully in Stage 6):
  In a distributed system, you can only guarantee 2 of 3:
  Consistency, Availability, Partition Tolerance

CP (Consistent + Partition Tolerant):
  PostgreSQL, MySQL, MongoDB (default config)
  Data is always consistent — if network partition, some nodes reject writes
  
AP (Available + Partition Tolerant):
  Cassandra, DynamoDB, CouchDB
  Always available — if network partition, may serve stale data
  Eventually consistent
  
CA (Consistent + Available — no partition tolerance):
  Only possible in single-node systems
  Not realistic for distributed systems
```

---

## SQL vs NoSQL — Side by Side

| Dimension | SQL | NoSQL |
|---|---|---|
| Schema | Fixed, enforced by DB | Flexible, enforced by app |
| Relationships | Native JOINs | Embedding or application-side |
| Transactions | Full ACID | Varies (Cassandra: partition-level, MongoDB: multi-doc in 4.0+) |
| Scale | Vertical + read replicas | Horizontal (designed for it) |
| Query flexibility | Extremely flexible — any query | Limited by data model |
| Consistency | Strong by default | Eventual by default |
| Data model | Tables, rows, columns | Documents, KV, columns, graphs |
| Schema changes | Migrations required | Usually non-breaking |
| Learning curve | Well understood, mature tooling | Varies by type |
| Best for | Complex relationships, transactions | Scale, flexibility, specific access patterns |

---

## Polyglot Persistence — Using Both in ShopSphere

The correct answer in most system design interviews is not "SQL or NoSQL" — it is "both, at different layers, for different purposes."

```
ShopSphere Polyglot Persistence Strategy:

PostgreSQL (SQL):
  ├── users            (accounts, auth — ACID critical)
  ├── orders           (financial records — transactions required)
  ├── order_items      (normalised, JOINed with orders)
  ├── payments         (financial — ACID non-negotiable)
  └── products         (core catalog — referential integrity)

MongoDB (Document):
  ├── product_catalog  (flexible schema — electronics ≠ clothing)
  ├── user_activity    (event logs — flexible payload)
  └── cms_content      (pages, banners — varied structure)

Redis (Key-Value):
  ├── sessions         (fast lookup, TTL)
  ├── rate_limits      (atomic counters)
  ├── cache            (product data, search results)
  └── cart             (fast read-write, temporary)

Cassandra (Column-Family):
  ├── order_history    (time-series, high write volume)
  └── audit_log        (append-only, time-ordered)

Elasticsearch (Inverted Index — covered in Stage 9):
  ├── product_search   (full-text search)
  └── order_search     (admin search with filters)

Neo4j (Graph):
  ├── recommendations  (collaborative filtering)
  └── fraud_detection  (relationship traversal)
```

**Decision framework for each ShopSphere entity:**
```
Placing an order:
  Need atomicity (payment + inventory + order record) → PostgreSQL

Searching for products:
  Need full-text search, facets, relevance scoring → Elasticsearch

Checking cart contents:
  Need fast read/write, temporary data, TTL → Redis

Querying order history for analytics:
  High write volume, time-ordered, no complex queries → Cassandra

Product recommendation:
  Relationship traversal across purchase graph → Neo4j

Product details page:
  Flexible schema (different product types), embedded specs → MongoDB
```

---

## Real-World Example — ShopSphere Data Layer

```java
// Order creation — multiple databases in one operation
@Service
public class OrderCreationService {

    @Autowired private OrderRepository postgresOrderRepo;        // PostgreSQL
    @Autowired private RedisTemplate<String, Cart> redisClient;  // Redis
    @Autowired private KafkaTemplate<String, OrderEvent> kafka;  // Kafka (event log)

    @Transactional  // PostgreSQL transaction
    public Order createOrder(String userId, CreateOrderRequest request) {

        // 1. Read cart from Redis
        Cart cart = redisClient.opsForValue().get("cart:" + userId);
        validateCart(cart, request);

        // 2. Persist order in PostgreSQL (ACID transaction)
        Order order = postgresOrderRepo.save(Order.builder()
            .userId(userId)
            .items(cart.getItems())
            .total(calculateTotal(cart))
            .status(OrderStatus.CONFIRMED)
            .build());

        // 3. Clear cart in Redis
        redisClient.delete("cart:" + userId);

        // 4. Publish event to Kafka (picked up by Cassandra audit writer,
        //    Elasticsearch indexer, Notification service, etc.)
        kafka.send("order-events", order.getId(),
            OrderConfirmedEvent.from(order));

        return order;
    }
}

// Separate service indexes into Elasticsearch for search
@KafkaListener(topics = "order-events", groupId = "search-indexer")
public void indexOrder(OrderConfirmedEvent event) {
    elasticsearchClient.index(i -> i
        .index("orders")
        .id(event.getOrderId())
        .document(OrderDocument.from(event)));
}

// Separate service writes to Cassandra for time-series history
@KafkaListener(topics = "order-events", groupId = "history-writer")
public void writeHistory(OrderConfirmedEvent event) {
    cassandraTemplate.insert(OrderHistoryEntry.from(event));
    // Stored by userId + timestamp — perfect for "my order history" queries
}
```

---

## Interview Q&A

**Q: When would you choose NoSQL over SQL?**
Choose NoSQL when your access patterns are well-defined and fixed — document stores like MongoDB excel when you always fetch a complete entity and never need to JOIN across entities. Choose column-family stores like Cassandra when you need massive write throughput and time-series ordering. Choose key-value stores like Redis for caching, sessions, and counters where O(1) lookup is paramount. The common thread is that NoSQL databases require you to design your data model around your access patterns upfront, and reward you with performance and horizontal scalability. SQL is better when you need flexible querying, complex relationships, and ACID transactions across entities.

**Q: What is polyglot persistence and when would you apply it?**
Polyglot persistence means using multiple different database technologies within the same system, choosing each one based on what it does best for that specific data and access pattern. In ShopSphere, PostgreSQL handles financial records and orders because ACID transactions are non-negotiable. Redis handles caching and sessions because O(1) lookup and TTL matter. Elasticsearch handles product search because full-text search and relevance scoring require an inverted index. Cassandra handles audit logs because they are append-only, time-ordered, and high-volume. The trade-off is operational complexity — more database technologies to operate, monitor, backup, and expertise to maintain.

**Q: What is the difference between a document store and a relational database for product catalogs?**
A relational database enforces a fixed schema across all products — every product has the same columns. A laptop and a T-shirt would both have columns for irrelevant attributes. Schema changes require migrations. A document store allows each product to have a completely different structure — a laptop document has CPU, RAM, and storage specs while a T-shirt document has size, colour, and material. Both coexist in the same collection with no migrations needed. The trade-off is that document stores sacrifice JOIN capability and referential integrity. For a product catalog where each category has unique attributes and reads vastly outnumber writes, the flexibility and read performance of a document store outweigh these costs.

**Q: Why can't you just use PostgreSQL for everything?**
PostgreSQL is excellent but has specific limitations at scale. Horizontal write scaling requires sharding which is complex to implement and breaks many relational guarantees. Full-text search is significantly less powerful than Elasticsearch. Time-series data at extremely high ingestion rates overwhelms relational row storage compared to column-family stores designed for it. Graph traversal queries involving deep relationship chains become exponentially slow with JOINs. Session storage and caching require network round trips to PostgreSQL when Redis serves the same data in sub-millisecond from memory. Using PostgreSQL for everything means accepting worse performance, higher cost, and higher complexity at scale in areas where specialised databases excel.

**Q: What does "schema on read" vs "schema on write" mean?**
Schema on write means the database enforces the schema when data is written — relational databases validate every row against the table schema at insert time. Invalid data is rejected. Schema on read means the database stores data without enforcing structure and the application interprets the schema when reading — document stores allow any shape of document to be written, and the application handles variable shapes at read time. Schema on write gives stronger consistency guarantees and catches data quality issues early. Schema on read gives more flexibility for evolving data structures but moves the responsibility for data quality to the application layer, which is harder to enforce and audit.

---

Say **"next"** when ready for Topic 8 — Object Storage.
