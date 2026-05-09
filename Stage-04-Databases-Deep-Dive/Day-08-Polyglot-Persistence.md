# Stage 4 — Databases Deep Dive
## Topic 8: Polyglot Persistence — Right Database for the Right Access Pattern

---

## 1. What Polyglot Persistence Is

The term was coined by Martin Fowler. The idea: **different services have different data needs — force-fitting everything into one database type is a mistake.**

A monolith era constraint was that you had one DB — usually Oracle or MySQL — and every feature bent itself to fit that model. Microservices break that constraint. Each service owns its data, so each service can choose the storage technology that best fits its access pattern.

```
One DB for everything (old way):
  All 22 ShopSphere services → PostgreSQL
  Result: sub-optimal for caching, search, time-series, graphs

Polyglot Persistence (right way):
  User/Order/Payment → PostgreSQL  (relational, ACID)
  Cart/Session       → Redis        (ephemeral, sub-ms)
  Search             → Elasticsearch (full-text, faceted)
  Analytics          → Cassandra    (write-heavy, time-series)
  Recommendations    → Neo4j        (graph traversal)
  Product Attributes → MongoDB      (flexible schema)
```

---

## 2. The Core Principle — Access Pattern First

Before choosing a DB, answer these questions about the service:

```
1. Read/Write ratio?
   Heavy reads → optimise for read speed (indexes, caching, replicas)
   Heavy writes → optimise for write throughput (LSM-tree, append-only)

2. Query pattern?
   Always by PK → Key-value or any DB (simple)
   By arbitrary fields → needs indexing (PostgreSQL, MongoDB)
   Full-text search → Elasticsearch
   Graph traversal → Neo4j
   Time-range on one entity → Cassandra

3. Data shape?
   Uniform rows → RDBMS
   Variable attributes → Document store
   Relationships between entities → Graph
   Flat key→value → Key-value store

4. Consistency requirement?
   Must be ACID → PostgreSQL
   Eventual OK → Cassandra, Redis
   Tunable per operation → Cassandra

5. Data lifetime?
   Permanent → persistent DB
   Ephemeral (sessions, carts) → Redis with TTL

6. Data volume and growth rate?
   Bounded → single PostgreSQL instance fine
   Unbounded writes (events, logs, metrics) → Cassandra / time-series DB
```

---

## 3. ShopSphere — Service by Service Decision

### User Service → PostgreSQL

**Access pattern:** Look up by email (login), look up by ID (FK joins), update profile, password reset.

**Why PostgreSQL:**
- Email uniqueness enforced at DB level (`UNIQUE` constraint)
- FK relationships to orders, reviews, addresses
- Low write volume (signups are rare vs order placements)
- ACID for password and auth operations

```sql
SELECT * FROM users WHERE email = 'v@x.com';  -- indexed lookup
UPDATE users SET password_hash = ? WHERE id = ?;  -- ACID
```

---

### Order Service → PostgreSQL

**Access pattern:** Place order (write), fetch order history (read by user_id), fetch single order (read by order_id), update order status.

**Why PostgreSQL:**
- Order placement is a multi-table transaction (orders + order_items + inventory)
- FK constraint to users and products
- `REPEATABLE READ` isolation during checkout
- Range partition by `created_at` handles data volume

---

### Payment Service → PostgreSQL (with synchronous replica)

**Access pattern:** Record payment, look up by order_id, audit trail queries.

**Why PostgreSQL (not anything else):**
- Zero tolerance for data loss — synchronous replication, fsync on every commit
- Idempotency keys stored with UNIQUE constraint
- Audit log is append-only but must be durable and queryable
- Regulatory requirement: ACID, no eventual consistency

---

### Cart Service → Redis

**Access pattern:** Get cart by user_id, add item, remove item, clear cart on checkout.

**Why Redis:**
- Cart is ephemeral — expires when user abandons or checks out
- Sub-millisecond reads (user clicks "Add to Cart", UX must feel instant)
- TTL built-in — `EXPIRE cart:u42 86400` (24h cart expiry, no cleanup job needed)
- Data structure: Redis Hash per cart

```
HSET cart:u42 product:p10 {"qty":2,"price":999}
HSET cart:u42 product:p55 {"qty":1,"price":299}
EXPIRE cart:u42 86400
```

**Why NOT PostgreSQL for cart:**
- Cart changes constantly (every add/remove is a write)
- Expired carts need cleanup (Redis TTL is automatic; DB needs a cron job)
- Cart loss on Redis failure is acceptable (user re-adds items); payment loss is not

---

### Session Service → Redis

**Access pattern:** Write session on login, read session on every authenticated request, delete on logout.

**Why Redis:**
- Every API request validates session → must be sub-millisecond
- Sessions expire naturally (TTL)
- No relational queries needed — pure key lookup

```
SET session:token_abc {"userId":"u42","roles":["USER"]} EX 3600
GET session:token_abc
DEL session:token_abc
```

---

### Search Service → Elasticsearch

**Access pattern:** Full-text search on product name/description, filter by category/price/brand, faceted counts ("512 results in Electronics"), autocomplete.

**Why Elasticsearch:**
- Inverted index — maps terms to document IDs; full-text is its native operation
- Faceted aggregations built in
- Relevance scoring (TF-IDF / BM25) — "laptop" query ranks gaming laptops over laptop bags
- Can't do this in PostgreSQL's `LIKE '%laptop%'` — no relevance, no facets, full table scan

**Data flow — dual-write via Kafka:**
```
Product Service (PostgreSQL) 
    → product update event → Kafka 
    → Search Consumer → Elasticsearch index
```
Elasticsearch is a read-optimised derived store — the source of truth stays in PostgreSQL.

---

### Notification Service → MongoDB

**Access pattern:** Write notification (variable schema per channel — email has subject/body, push has title/badge, SMS has text), read unread notifications by user, mark as read.

**Why MongoDB:**
```json
// Email notification
{"userId":"u42","channel":"EMAIL","subject":"Order Confirmed","body":"...","read":false}

// Push notification  
{"userId":"u42","channel":"PUSH","title":"Shipped!","badge":1,"deeplink":"/orders/123","read":false}

// SMS
{"userId":"u42","channel":"SMS","text":"Your OTP is 482910","read":true}
```
Variable fields per channel type → document model is natural. PostgreSQL would need nullable columns or a JSONB field (gives up type safety).

---

### Analytics / Event Ingestion → Cassandra

**Access pattern:** Write millions of events per second (page views, clicks, add-to-carts, searches), read by user_id + time range for user behaviour analysis.

**Why Cassandra:**
- LSM-Tree handles millions of writes/second without bottleneck
- Partition by `user_id`, cluster by `event_time` → "all events for user u42 in January" is a single-partition sequential read
- No single point of failure — critical for event ingestion (can't drop events)
- TTL on events (keep 1 year, auto-delete older)

```cql
CREATE TABLE user_events (
  user_id   UUID,
  event_time TIMESTAMP,
  event_type TEXT,
  metadata   MAP<TEXT, TEXT>,
  PRIMARY KEY (user_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND default_time_to_live = 31536000;  -- 1 year TTL
```

---

### Recommendation Service → Neo4j

**Access pattern:** "Users who bought X also bought Y", "Products frequently bought together", "Users similar to you liked Z."

**Why Neo4j:**
```cypher
// Products frequently co-purchased with product p10
MATCH (p1:Product {id: "p10"})<-[:PURCHASED]-(u:User)-[:PURCHASED]->(p2:Product)
WHERE p2.id <> "p10"
RETURN p2.name, COUNT(u) AS co_buyers
ORDER BY co_buyers DESC
LIMIT 5
```
This multi-hop traversal (Product → Users → Other Products) is natural in a graph. In PostgreSQL it's 2 joins on potentially billions of order_items rows — catastrophically slow.

---

## 4. The Operational Cost of Polyglot Persistence

This is the honest flip side. More DB types = more to operate.

| Cost | Description |
|---|---|
| **Ops overhead** | Each DB type needs its own monitoring, backup strategy, tuning expertise, and on-call runbook |
| **Consistency across DBs** | When product data changes in PostgreSQL, Elasticsearch must be updated. Dual-write bugs create divergence. |
| **No cross-DB joins** | "Orders with product name from Elasticsearch" — impossible at DB level. Must join in application layer. |
| **Transaction boundaries** | A transaction cannot span PostgreSQL and MongoDB. Sagas or compensating logic required. |
| **Team knowledge** | Dev team must understand Redis, Cassandra, Elasticsearch, MongoDB — breadth vs depth trade-off. |
| **Data consistency bugs** | Kafka consumer fails mid-update — Elasticsearch is stale. Need idempotent consumers, dead-letter queues, reconciliation jobs. |

### The Mitigation

**Don't add a DB type unless the access pattern genuinely justifies it.** PostgreSQL with the right indexes and read replicas handles enormous scale. Elasticsearch is justified for full-text search. Redis is justified for caching and sessions. Cassandra is justified for millions of writes/second. Neo4j is justified only if graph traversal is a core product feature.

A startup with 10K users should use PostgreSQL for everything and optimise from there. A company at ShopSphere's target scale makes these choices deliberately.

---

## 5. Data Synchronisation Patterns

When the same data lives in multiple stores, keeping them in sync is the hardest part.

### Pattern 1: Kafka as the Synchronisation Bus

```
PostgreSQL (source of truth)
    → Debezium CDC (reads WAL, emits change events)
    → Kafka topic: product-changes
    → Elasticsearch consumer (updates search index)
    → Analytics consumer (updates Cassandra)
    → Recommendation consumer (updates Neo4j edges)
```

Debezium reads PostgreSQL WAL directly — no dual-write in application code. If a consumer fails, it replays from Kafka. Eventually consistent but reliable.

### Pattern 2: Outbox Pattern

Application writes to PostgreSQL + an outbox table in the same transaction. A relay process reads the outbox and publishes to Kafka. Guarantees: if the DB write succeeded, the event will eventually be published.

### Pattern 3: Cache-Aside

```
Read: 
  Check Redis → hit? return. miss? read PostgreSQL → write to Redis → return.

Write:
  Write to PostgreSQL → invalidate Redis key (not update — invalidate).
  Next read will repopulate from PostgreSQL.
```

---

## 6. Interview Q&A

**Q: What is polyglot persistence and what problem does it solve?**
A: Polyglot persistence is the practice of using different database technologies for different services based on each service's specific access pattern, consistency needs, and data shape. It solves the mismatch between a one-size-fits-all database and the diverse needs of a microservices system — caching needs sub-millisecond key lookup (Redis), search needs inverted indexes and relevance scoring (Elasticsearch), analytics needs high write throughput (Cassandra), and core transactional data needs ACID guarantees (PostgreSQL).

**Q: What are the downsides of polyglot persistence?**
A: Operational complexity — each DB type needs separate expertise, monitoring, backup, and incident runbooks. Cross-DB consistency is hard — data changes in PostgreSQL must propagate to Elasticsearch, Redis, and Cassandra, and any failure in that pipeline creates divergence. No cross-DB transactions — workflows spanning multiple stores need Sagas with compensating logic. Team knowledge breadth required increases. The rule: only add a new DB type when the access pattern genuinely cannot be served well by existing stores.

**Q: How do you keep Elasticsearch in sync with PostgreSQL?**
A: Two main approaches. CDC (Change Data Capture) with Debezium — Debezium reads PostgreSQL's WAL, emits row-level change events to Kafka, and an Elasticsearch consumer applies them. This is the cleanest approach — no application code changes, reliable ordering via Kafka partitions. Alternative: Outbox pattern — application writes a sync event to an outbox table in the same transaction as the main write, a relay reads the outbox and publishes to Kafka, Elasticsearch consumer applies. Both approaches are eventually consistent — Elasticsearch lags PostgreSQL by seconds.

**Q: When would you NOT use Redis as a cache?**
A: When data is too large to fit in RAM affordably. When cache invalidation logic is too complex and staleness would cause correctness bugs (e.g., financial balances — better to read from primary always). When the underlying query is fast enough that caching overhead doesn't pay off. When data changes so frequently that the cache hit rate is too low to be useful. Redis shines for data that's read far more than it's written and where slight staleness is acceptable.

**Q: Your Elasticsearch index is out of sync with PostgreSQL — how do you detect and fix it?**
A: Detection: run a reconciliation job periodically — compare document counts and spot-check random records between PostgreSQL and Elasticsearch. Also monitor Kafka consumer lag — if the consumer is behind, the index is stale by that lag amount. Fix: for minor drift, replay Kafka events from a checkpoint. For severe drift or corruption, trigger a full reindex — read all products from PostgreSQL in batches and bulk-index into Elasticsearch. This is why keeping PostgreSQL as the source of truth matters — Elasticsearch is a derived read store that can always be rebuilt.

---

Ready for **Topic 9: Connection Pooling — deep dive into internals, sizing, PgBouncer, and failure modes**?
