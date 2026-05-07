# Stage 3 — Topic 11: Different Storage Systems

## Theory

Every system design interview eventually requires you to choose where data lives. The mistake most candidates make is defaulting to one or two databases they know well — PostgreSQL and Redis — and treating every storage problem as a nail for those hammers.

The reality is that modern systems use a **portfolio of storage technologies**, each optimised for a specific data shape, access pattern, and performance characteristic. This topic builds a complete taxonomy of every storage system you need to know — what it is, why it exists, what trade-offs it makes, and when to reach for it.

**The organising principle:**
```
Every storage system makes a fundamental trade-off between:

  Write speed    vs  Read speed
  Consistency    vs  Availability
  Query flexibility vs  Performance
  Storage cost   vs  Access speed
  General purpose vs  Specialised

No system optimises all of these simultaneously.
Choosing the right storage = matching the trade-off to the problem.
```

---

## The Complete Storage Taxonomy

```
Storage Systems
├── Relational (SQL)
│   ├── PostgreSQL
│   ├── MySQL
│   └── SQLite
│
├── Key-Value Stores
│   ├── Redis (in-memory + persistence)
│   └── Memcached (pure in-memory)
│
├── Document Stores
│   ├── MongoDB
│   └── CouchDB
│
├── Column-Family (Wide-Column)
│   ├── Apache Cassandra
│   └── HBase
│
├── Graph Databases
│   ├── Neo4j
│   └── Amazon Neptune
│
├── Search Engines
│   ├── Elasticsearch
│   └── Apache Solr
│
├── Time-Series Databases
│   ├── InfluxDB
│   ├── TimescaleDB (PostgreSQL extension)
│   └── Prometheus (metrics only)
│
├── Object Storage
│   ├── AWS S3
│   ├── Google Cloud Storage
│   └── MinIO (self-hosted)
│
├── Block Storage
│   ├── AWS EBS
│   └── Local NVMe SSD
│
├── File Storage
│   ├── AWS EFS
│   └── NFS
│
├── Data Warehouses (OLAP)
│   ├── Amazon Redshift
│   ├── Google BigQuery
│   └── Snowflake
│
└── NewSQL (SQL + Horizontal Scale)
    ├── CockroachDB
    ├── Google Spanner
    └── TiDB
```

---

## Category 1 — Relational Databases

Already covered in depth in Topic 7. Summary:

```
PostgreSQL — the production workhorse:
  ACID transactions across multiple tables
  Rich SQL — window functions, CTEs, JSON support
  MVCC concurrency — readers don't block writers
  Extensions: PostGIS (geospatial), TimescaleDB, pgvector
  Scales vertically + read replicas
  Best for: orders, payments, users, anything requiring transactions

MySQL / MariaDB:
  Similar to PostgreSQL, slightly different trade-offs
  InnoDB engine: ACID, row-level locking
  Widely used in web applications (WordPress, Shopify historically)
  Replication: standard primary-replica, Group Replication, Galera Cluster
  Best for: web applications, read-heavy workloads

SQLite:
  Serverless — the entire DB is one file
  No separate process — embedded in the application
  Perfect for: mobile apps, desktop apps, testing, small scripts
  Not for: concurrent writes, distributed systems, high throughput
```

---

## Category 2 — Key-Value Stores

### Redis — In Depth

```
Architecture:
  Single-threaded event loop (I/O operations multithreaded in Redis 6+)
  All data in RAM — operations are nanosecond to microsecond
  Optional persistence: RDB snapshots + AOF write-ahead log
  
Data structures (this is Redis's superpower):

String:   SET key value EX 3600
          GET key
          INCR counter           ← atomic increment
          
Hash:     HSET user:u-123 name "Vidhan" email "v@e.com"
          HGET user:u-123 name
          HGETALL user:u-123
          Use: user sessions, object caching, partial updates
          
List:     LPUSH queue job1        ← prepend
          RPOP queue              ← dequeue (FIFO queue)
          LRANGE list 0 -1        ← get all
          Use: message queues, activity feeds, recent items

Set:      SADD tags "shoes" "nike" "sports"
          SISMEMBER tags "nike"   ← O(1) membership check
          SINTER set1 set2        ← intersection
          SUNION set1 set2        ← union
          Use: unique visitors, tags, friend lists

Sorted Set: ZADD leaderboard 1299.99 "seller-001"
            ZREVRANGE leaderboard 0 9   ← top 10
            ZRANK leaderboard "seller-001"  ← rank of seller
            Use: leaderboards, rate limiting sliding window,
                 priority queues, time-series (score=timestamp)

Streams:  XADD events * type ORDER_CONFIRMED orderId o-789
          XREAD COUNT 10 STREAMS events 0
          Use: event log, pub/sub with persistence, activity feeds
          Similar to Kafka but simpler, in-memory-first

Geospatial: GEOADD locations 73.8567 18.5204 "pune"
            GEOSEARCH locations FROMMEMBER "pune" BYRADIUS 100 km ASC
            Use: nearby seller search, delivery radius

HyperLogLog: PFADD unique_visitors user1 user2 user3
             PFCOUNT unique_visitors   ← approximate unique count
             Uses 12KB regardless of cardinality
             Use: unique visitor counts, distinct query counts
```

**Redis persistence modes:**
```
No persistence (default for pure cache):
  Fastest — all writes to RAM only
  All data lost on restart
  Use: ephemeral cache, sessions with acceptable loss

RDB (Redis Database — snapshot):
  Periodic fork + dump to disk
  BGSAVE every N seconds or M changes (configurable)
  Fast restore — load snapshot on startup
  Data loss: up to last snapshot interval
  
AOF (Append-Only File):
  Every write command appended to log file
  Replay log on restart — reconstruct full state
  fsync options: always, everysec, no
    always:   every write flushed — slowest, no data loss
    everysec: flush every second — good balance
    no:       OS decides — fastest, up to 1 second loss
  
RDB + AOF:
  Both enabled — best durability
  Use RDB for fast restores, AOF for minimal data loss
  Production recommendation for ShopSphere
```

---

## Category 3 — Document Stores

### MongoDB — Key Internals

```
Storage engine: WiredTiger (since MongoDB 3.2)
  MVCC — readers and writers don't block each other
  Document-level locking (not collection-level)
  Compression: Snappy (default), zlib, zstd

BSON format:
  Binary JSON — more types than JSON
  int32, int64, double, decimal128, date, ObjectId, binary
  Size limit: 16MB per document

ObjectId:
  12-byte auto-generated primary key
  4 bytes: Unix timestamp
  5 bytes: random process identifier
  3 bytes: auto-incrementing counter
  Roughly time-ordered — useful for sorting by insertion time

Replica Set:
  1 Primary + N Secondaries
  Primary: all writes
  Secondaries: reads (if readPreference allows) + failover
  Automatic failover: secondary elected primary in ~10 seconds

Sharding:
  Horizontal partitioning across multiple shards
  Shard key choice is critical — bad key = hot shard problem
  Range sharding vs Hash sharding
  ShardConfig servers manage metadata
```

**MongoDB indexes:**
```javascript
// Single field index
db.products.createIndex({ "price": 1 })  // 1=ascending, -1=descending

// Compound index — order matters
db.products.createIndex({ "category": 1, "price": 1 })
// Supports: { category: "shoes" }
// Supports: { category: "shoes", price: { $gt: 50 } }
// Does NOT support: { price: { $gt: 50 } } alone

// Text index — full text search
db.products.createIndex({ "description": "text", "name": "text" })
db.products.find({ $text: { $search: "running shoes" } })

// TTL index — auto-delete expired documents
db.sessions.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 86400 })

// Partial index — only index documents matching filter
db.orders.createIndex(
    { "userId": 1 },
    { partialFilterExpression: { status: "CONFIRMED" } }
)
// Only indexes confirmed orders — smaller, faster
```

---

## Category 4 — Column-Family Stores

### Cassandra — LSM Tree Write Path

```
Why Cassandra is fast for writes:
  Uses LSM-Tree (Log-Structured Merge Tree)
  
Write path:
  1. Write to CommitLog (WAL) on disk — sequential write, fast
  2. Write to Memtable (in-memory sorted structure)
  3. Return success to client — write complete in ~1ms
  
  Background:
  4. When Memtable full → flush to SSTable on disk (immutable)
  5. Compaction: merge multiple SSTables → remove tombstones,
     deduplicate, sort — keeps read performance reasonable

Read path:
  1. Check Bloom filter — "is this key possibly in this SSTable?"
     (probabilistic — can say "definitely not", not "definitely yes")
  2. Check Memtable — data might be there (newest)
  3. Check SSTables (newest first) — stop when found
  
  More SSTables = more reads = compaction keeps this bounded

LSM Tree trade-off:
  Writes: fast — sequential append to log + in-memory
  Reads:  potentially slower — must check multiple SSTables
  
Opposite of B-Tree (PostgreSQL):
  B-Tree: reads fast (O(log n) tree traversal), writes slower (page updates)
  LSM:    writes fast (sequential append), reads potentially slower
  
This is why:
  Cassandra: write-heavy time-series, IoT, audit logs
  PostgreSQL: read-heavy transactional, complex queries
```

**Cassandra data modelling — access pattern first:**
```cql
-- Design rule: ONE table per query pattern

-- Query 1: Get orders for a user, newest first
CREATE TABLE orders_by_user (
    user_id     UUID,
    created_at  TIMESTAMP,
    order_id    UUID,
    status      TEXT,
    total       DECIMAL,
    PRIMARY KEY ((user_id), created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Query 2: Get orders by status (admin view)
CREATE TABLE orders_by_status (
    status      TEXT,
    created_at  TIMESTAMP,
    order_id    UUID,
    user_id     UUID,
    total       DECIMAL,
    PRIMARY KEY ((status), created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Same data, two tables — data duplication is intentional and required
-- Each table optimised for exactly one access pattern

-- Query 3: Get order by ID
CREATE TABLE orders_by_id (
    order_id    UUID,
    user_id     UUID,
    status      TEXT,
    total       DECIMAL,
    created_at  TIMESTAMP,
    PRIMARY KEY (order_id)
);
```

---

## Category 5 — Search Engines

### Elasticsearch — How It Works

```
Core data structure: Inverted Index

Document: "Nike Air Max running shoes for men"

Inverted Index:
  "nike"    → [doc1, doc5, doc12]
  "air"     → [doc1, doc7]
  "max"     → [doc1]
  "running" → [doc1, doc3, doc8, doc15]
  "shoes"   → [doc1, doc3, doc4, doc9]
  "men"     → [doc1, doc2, doc6]

Query: "running shoes"
  → documents containing "running": [doc1, doc3, doc8, doc15]
  → documents containing "shoes":   [doc1, doc3, doc4, doc9]
  → intersection + relevance score:  doc1, doc3 (highest relevance)

This is O(1) per term lookup regardless of corpus size
vs SQL LIKE '%running shoes%' = full table scan = O(n)
```

**Elasticsearch concepts:**
```
Index:     Collection of documents (like a DB table)
Document:  JSON object (like a DB row)
Field:     JSON key (like a DB column)
Shard:     Partition of an index — enables horizontal scaling
Replica:   Copy of a shard — enables HA and read scaling
Cluster:   Multiple nodes sharing workload

Mapping:
  Schema for an index — field types
  text:     full-text searchable, analysed
  keyword:  exact match, not analysed (for filtering, aggregations)
  integer, float, date, boolean, geo_point

Analyser:
  How text is processed at index and query time
  Standard: lowercase, tokenise on whitespace
  Custom: stemming (running→run), stop words, synonyms
```

```java
// ShopSphere — Product search with Elasticsearch
@Repository
public class ProductSearchRepository {

    @Autowired
    private ElasticsearchClient elasticsearchClient;

    public SearchResult<ProductDocument> search(ProductSearchRequest request) {

        SearchResponse<ProductDocument> response = elasticsearchClient.search(s -> s
            .index("products")
            .query(q -> q
                .bool(b -> b
                    // Full text search on name and description
                    .must(m -> m
                        .multiMatch(mm -> mm
                            .query(request.getQuery())
                            .fields("name^3", "description", "tags")
                            // ^3 = name matches worth 3x description matches
                        )
                    )
                    // Filter — exact match, doesn't affect score
                    .filter(f -> f
                        .term(t -> t.field("category").value(request.getCategory()))
                    )
                    .filter(f -> f
                        .range(r -> r
                            .field("price")
                            .gte(JsonData.of(request.getMinPrice()))
                            .lte(JsonData.of(request.getMaxPrice()))
                        )
                    )
                    .filter(f -> f
                        .term(t -> t.field("inStock").value(true))
                    )
                )
            )
            // Aggregations for faceted search
            .aggregations("categories", a -> a
                .terms(t -> t.field("category").size(20))
            )
            .aggregations("priceRanges", a -> a
                .range(r -> r
                    .field("price")
                    .ranges(
                        range -> range.to(50.0),
                        range -> range.from(50.0).to(200.0),
                        range -> range.from(200.0).to(500.0),
                        range -> range.from(500.0)
                    )
                )
            )
            // Sorting
            .sort(sort -> sort
                .field(f -> f
                    .field(request.getSortBy())  // "price", "rating", "_score"
                    .order(request.getSortOrder().equals("asc")
                        ? SortOrder.Asc : SortOrder.Desc)
                )
            )
            .from(request.getPage() * request.getSize())
            .size(request.getSize()),
            ProductDocument.class
        );

        return buildSearchResult(response);
    }
}
```

---

## Category 6 — Time-Series Databases

### What Makes Time-Series Different

```
Time-series data characteristics:
  High write throughput — millions of data points per second
  Always append — never update historical data
  Time-ordered — queries almost always involve time ranges
  Data expires — old data less valuable, should auto-delete
  Compression — similar consecutive values compress extremely well

Traditional databases struggle with time-series:
  PostgreSQL: index becomes huge, writes slow as table grows
  One week of metrics at 1M datapoints/second = 604 billion rows
  B-tree index on timestamp = constant rebalancing overhead
  
Time-series databases solve this with:
  Time-based partitioning — chunks by time period
  Column-oriented storage — similar values per column compress 10-100x
  Automatic data retention — delete old chunks efficiently
  Downsampling — compress old data to lower resolution
```

### InfluxDB

```
InfluxDB concepts:
  Measurement: like a table (e.g. "api_requests")
  Tag:         indexed metadata for filtering (e.g. service="order-service")
  Field:       the actual measured value (e.g. response_time=45.2)
  Timestamp:   nanosecond precision

Write:
  api_requests,service=order-service,endpoint=/orders response_time=45.2,status=200 1712566200000000000

Query (Flux language):
  from(bucket: "shopsphere-metrics")
    |> range(start: -1h)
    |> filter(fn: (r) => r.service == "order-service")
    |> filter(fn: (r) => r._measurement == "api_requests")
    |> aggregateWindow(every: 1m, fn: mean)
    |> yield()

  → Average response time per minute for last hour
```

### TimescaleDB — PostgreSQL Extension

```
TimescaleDB extends PostgreSQL with:
  Hypertables — automatically partitioned by time
  Continuous aggregates — precomputed rollups (materialised views that auto-refresh)
  Data retention policies — auto-delete old partitions
  Compression — columnar storage for old chunks

Benefit: use standard SQL + time-series optimisations
  SELECT
      time_bucket('1 minute', created_at) AS bucket,
      AVG(response_time) AS avg_response,
      COUNT(*) AS request_count
  FROM api_requests
  WHERE created_at >= NOW() - INTERVAL '1 hour'
    AND service = 'order-service'
  GROUP BY bucket
  ORDER BY bucket;
```

### Prometheus — Metrics Specifically

```
Prometheus is NOT a general time-series DB.
It is specifically for operational metrics.

Architecture:
  Prometheus scrapes metrics from targets every N seconds
  (Pull model — targets expose /metrics endpoint)
  Stores in local time-series database
  
Metric types:
  Counter:   monotonically increasing (total requests, total errors)
  Gauge:     value that goes up and down (memory usage, active connections)
  Histogram: distribution of values (request latency distribution)
  Summary:   similar to histogram, pre-calculated quantiles

ShopSphere metrics endpoint (Spring Actuator + Micrometer):
  GET /actuator/prometheus
  
  # HELP http_requests_total Total HTTP requests
  # TYPE http_requests_total counter
  http_requests_total{method="GET",uri="/orders",status="200"} 15432.0
  http_requests_total{method="POST",uri="/orders",status="201"} 2341.0
  http_requests_total{method="POST",uri="/orders",status="400"} 45.0

  # HELP http_request_duration_seconds Request duration
  # TYPE http_request_duration_seconds histogram
  http_request_duration_seconds_bucket{le="0.05"} 8432.0
  http_request_duration_seconds_bucket{le="0.1"}  12341.0
  http_request_duration_seconds_bucket{le="0.5"}  15200.0
  http_request_duration_seconds_bucket{le="+Inf"} 15432.0

Query (PromQL):
  rate(http_requests_total{status="500"}[5m])
  → Per-second error rate over last 5 minutes
  
  histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  → p99 latency over last 5 minutes
```

---

## Category 7 — Data Warehouses (OLAP)

### OLTP vs OLAP — The Fundamental Split

```
OLTP (Online Transactional Processing):
  Optimised for: individual row operations
  Query pattern: INSERT, UPDATE, DELETE + simple SELECTs by primary key
  Example: "Get order o-789" — one row, fast
  Database: PostgreSQL, MySQL
  Row-oriented storage: fast for full-row reads

OLAP (Online Analytical Processing):
  Optimised for: aggregate queries across millions of rows
  Query pattern: complex aggregations across entire dataset
  Example: "Revenue by product category for Q1 2026"
  Database: Redshift, BigQuery, Snowflake
  Column-oriented storage: fast for reading one column across many rows

Why you NEVER run analytics queries on your OLTP database:
  "SELECT category, SUM(total) FROM orders WHERE..."
  → Full table scan on orders table
  → Locks rows, slows down order creation transactions
  → As data grows: query takes minutes
  → Users cannot place orders while query runs
  
  This is why ShopSphere ETLs order data to Redshift/BigQuery
  for analytics — never queries production PostgreSQL for reports
```

### Column-Oriented Storage — Why OLAP Is Fast

```
Row-oriented (PostgreSQL):
  Row 1: [id=1, name="Nike", price=99.99, category="shoes", sold=1000]
  Row 2: [id=2, name="Adidas", price=79.99, category="shoes", sold=800]
  
  Stored: [1, Nike, 99.99, shoes, 1000, 2, Adidas, 79.99, shoes, 800...]
  
  Query: SELECT SUM(sold) FROM products WHERE category='shoes'
  → Must read every row including id, name, price — not needed
  → IO wasted reading irrelevant columns

Column-oriented (BigQuery):
  id column:       [1, 2, 3, 4, 5...]
  name column:     [Nike, Adidas, Puma...]
  price column:    [99.99, 79.99, 89.99...]
  category column: [shoes, shoes, clothing...]
  sold column:     [1000, 800, 1200...]
  
  Query: SELECT SUM(sold) WHERE category='shoes'
  → Read only 'category' and 'sold' columns
  → 2 columns instead of 5 = 60% less IO
  → Columns compress extremely well (similar values adjacent)
  → 1000, 800, 1200 → delta encoded: 1000, -200, +400 = tiny
  
  1 billion rows query: seconds instead of hours
```

### Google BigQuery — Serverless OLAP

```
Fully managed — no cluster to manage
Separates storage from compute
Pay per query (bytes scanned) — $5 per TB scanned

ShopSphere analytics pipeline:
  Kafka → Kafka Connect → BigQuery
  (orders, payments, events stream in continuously)

Query example:
  SELECT
      FORMAT_DATE('%Y-%m', DATE(created_at)) AS month,
      category,
      SUM(total) AS revenue,
      COUNT(*) AS order_count,
      AVG(total) AS avg_order_value
  FROM `shopsphere.orders`
  JOIN `shopsphere.order_items` USING (order_id)
  JOIN `shopsphere.products` USING (product_id)
  WHERE created_at >= '2026-01-01'
  GROUP BY month, category
  ORDER BY month, revenue DESC;
  
  → Scans 100GB of data
  → Returns in ~3 seconds
  → Cost: $0.50
  → Would take hours and cripple PostgreSQL
```

---

## Category 8 — NewSQL

### The Problem NewSQL Solves

```
The scale dilemma:

SQL (PostgreSQL):
  ✅ ACID transactions
  ✅ Flexible queries
  ❌ Hard to scale horizontally (sharding breaks transactions)

NoSQL (Cassandra):
  ✅ Horizontal scale — linear
  ✅ High availability
  ❌ No transactions across partitions
  ❌ Limited query flexibility

NewSQL — best of both worlds:
  ✅ SQL and ACID transactions
  ✅ Horizontal scale — like NoSQL
  ✅ Distributed — no single point of failure
  ❌ Higher operational complexity
  ❌ Higher latency than single-node PostgreSQL
```

### CockroachDB

```
Distributed SQL — ACID transactions across distributed nodes
Inspired by Google Spanner

Key features:
  Distributed transactions: Raft consensus per range
  Geo-partitioning: pin data to specific regions (GDPR compliance)
  Serializable isolation: strongest isolation level
  PostgreSQL-compatible wire protocol — use existing drivers

Architecture:
  Data split into ranges (64MB each)
  Each range replicated 3x across nodes via Raft
  Transactions use a 2-phase commit variant across ranges
  Timestamp ordering ensures serializability

Trade-offs:
  Higher latency than PostgreSQL (cross-node coordination)
  Best when: ACID + horizontal scale + multi-region required
  Overkill for: single-region apps that can scale with read replicas

ShopSphere consideration:
  If expanding globally with data residency requirements
  CockroachDB lets Indian users' data stay in India
  while still having globally consistent transactions
```

### Google Spanner

```
Google's globally distributed SQL database
Powers Google Ads, Gmail, Google Play

TrueTime API:
  GPS + atomic clocks in every datacenter
  Provides bounded uncertainty on wall clock
  Enables globally consistent transactions without central coordinator
  
  Commit timestamp = current time ± 7ms uncertainty
  Transactions wait out the uncertainty window
  Result: external consistency — global serializability
  
Capabilities:
  SQL with JOINs across globally distributed data
  ACID transactions spanning continents
  99.999% availability SLA
  Automatic sharding and rebalancing
  
Cost: extremely expensive — used by large enterprises
Available as: Google Cloud Spanner ($0.30/node-hour)
```

---

## Storage System Decision Framework

```
New data storage requirement — ask these questions:

Q1: Is the data relational with complex JOIN requirements?
    YES → PostgreSQL / MySQL

Q2: Is it a simple key-value with TTL (cache, session, counter)?
    YES → Redis

Q3: Is it flexible schema, document-shaped, embed-friendly?
    YES → MongoDB

Q4: Is it high-write-throughput, time-ordered, append-only?
    YES → Cassandra / InfluxDB

Q5: Is it full-text searchable with relevance scoring needed?
    YES → Elasticsearch

Q6: Is it large binary data (images, video, files, backups)?
    YES → S3 / Object Storage

Q7: Is it analytical (aggregate queries over millions of rows)?
    YES → BigQuery / Redshift / Snowflake

Q8: Is it relationship-graph data (social graph, fraud, recommendations)?
    YES → Neo4j

Q9: Is it operational metrics with time-based retention?
    YES → Prometheus + InfluxDB

Q10: Is it SQL requirements + horizontal scale + multi-region?
     YES → CockroachDB / Spanner
```

---

## Real-World Example — ShopSphere Complete Storage Map

```
Every piece of data in ShopSphere and where it lives:

User data:
  user accounts, passwords, roles    → PostgreSQL (ACID, integrity)
  session tokens                     → Redis (TTL, fast lookup)
  user preferences                   → MongoDB (flexible schema)
  user activity log                  → Cassandra (time-series append)

Product data:
  product records, inventory         → PostgreSQL (transactions)
  product catalog, specs             → MongoDB (flexible schema per type)
  product search index               → Elasticsearch (full-text)
  product images, videos             → S3 (object storage)
  image cache                        → Redis + CDN

Order data:
  orders, payments, order items      → PostgreSQL (ACID critical)
  order history per user             → Cassandra (time-series, read by userId+time)
  order search (admin)               → Elasticsearch

Analytics:
  operational metrics                → Prometheus + InfluxDB
  business analytics                 → BigQuery (revenue, cohorts, funnels)
  real-time counters (views, clicks) → Redis (INCR)
  raw event stream                   → Kafka (retained 7 days)

Infrastructure:
  application cache                  → Redis
  distributed locks                  → Redis (SET NX)
  rate limiting                      → Redis (sliding window counter)
  pub/sub backplane                  → Redis
  message broker                     → Kafka (events) + RabbitMQ (tasks)
  logs                               → S3 (archived) + Elasticsearch (searchable)
  config/secrets                     → HashiCorp Vault / AWS Secrets Manager

Relationships:
  seller-product-category graph      → Neo4j
  recommendations                    → Neo4j
  fraud detection graph              → Neo4j
```

---

## Interview Q&A

**Q: How would you choose between PostgreSQL and Cassandra for storing order history?**
Order history has two different access patterns — transactional operations like placing an order need ACID guarantees and benefit from PostgreSQL's relational model and JOIN support. Historical reads like "show me all orders for user u-123 from the last 6 months" are time-ordered, append-heavy, and need to scale with user growth without affecting write performance. A common pattern is to use PostgreSQL as the source of truth for active orders and stream confirmed order events to Cassandra optimised with userId as the partition key and created_at as the clustering key. This separates the transactional workload from the historical read workload, each scaled independently.

**Q: What is the difference between OLTP and OLAP and why should they never share a database?**
OLTP handles individual transactional operations — insert an order, update a status, fetch a user — with small, fast queries touching a few rows. OLAP handles complex analytical aggregations — revenue by category for the quarter, customer cohort analysis, funnel conversion rates — scanning millions of rows with heavy joins and groupings. Running OLAP queries on an OLTP database causes full table scans that acquire locks, slow down writes, and degrade the transactional performance real users experience. OLAP databases use column-oriented storage which reads only the columns needed, compresses similar values efficiently, and can process billion-row aggregations in seconds without affecting transactional systems.

**Q: When would you use Elasticsearch vs PostgreSQL for search?**
PostgreSQL full-text search works for basic keyword matching but lacks relevance scoring, tokenisation, stemming, synonyms, and faceted aggregation. A query like "Nike running shoes" on PostgreSQL is a LIKE scan or tsvector match — fast enough for small datasets but no relevance ranking, no typo tolerance, no autocomplete. Elasticsearch uses an inverted index which makes O(1) term lookups regardless of corpus size, scores results by relevance using TF-IDF or BM25, supports aggregations for faceted navigation — filter by category, price range, brand simultaneously — and handles text analysis like stemming running to run and handling misspellings. For a product search with millions of items, Elasticsearch is the correct choice. Keep PostgreSQL as the source of truth and sync to Elasticsearch via events.

**Q: What is a time-series database and why not just use PostgreSQL for metrics?**
A time-series database is optimised specifically for data that arrives continuously and is always associated with a timestamp — application metrics, IoT sensor readings, financial tick data. PostgreSQL stores data in row-oriented pages — writing millions of metric rows per second requires updating B-tree indexes on every insert, causing write amplification and index bloat. Time-series databases use column-oriented storage with time-based chunking — appending new time chunks is sequential, old chunks are compressed with delta encoding achieving 10 to 100x compression, and retention policies delete old chunks by simply dropping files rather than running deletes. TimescaleDB extends PostgreSQL with these capabilities, giving the best of both worlds for teams already using PostgreSQL.

**Q: What is NewSQL and when would ShopSphere need it?**
NewSQL databases like CockroachDB and Google Spanner provide SQL semantics and ACID transactions with horizontal scalability across distributed nodes — solving the dilemma where SQL cannot scale horizontally and NoSQL cannot provide transactions. ShopSphere would need NewSQL if it expanded globally with data residency requirements — GDPR mandates that European users' personal data stays in Europe, but orders referencing those users need to be consistent globally. CockroachDB's geo-partitioning pins specific rows to specific regions while maintaining globally consistent transactions. This is overkill for a single-region deployment where PostgreSQL with read replicas handles the scale, but becomes necessary at global scale with regulatory constraints.

---

