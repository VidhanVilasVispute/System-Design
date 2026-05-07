# Stage 4 — Databases Deep Dive
## Topic 7: NoSQL Categories — Key-Value, Document, Column-Family, Graph

---

## 1. Why NoSQL Exists

RDBMS gives you ACID, joins, constraints, and a flexible query model. But it makes hard assumptions:

- Schema is defined upfront and is rigid
- Data fits in tables with rows and columns
- Scaling writes means sharding — painful
- Every access pattern goes through SQL

NoSQL databases make the **opposite trade** — they sacrifice generality (no joins, no arbitrary queries, relaxed consistency) to gain:

- Horizontal write scalability built in
- Schema flexibility
- Data models optimised for specific access patterns
- Predictable low-latency at scale

"NoSQL" doesn't mean "no SQL syntax" — it means **Not Only SQL**. There are four major categories, each with a fundamentally different data model.

---

## 2. Key-Value Stores — Redis

### Data Model

The simplest possible model: a giant distributed hashmap.

```
key           →   value (opaque blob)
"session:abc" →   {"userId": "u42", "cart": [...], "expiry": 1720000000}
"rate:ip:1.2.3.4" →  "47"
"product:501" →   "{...full product JSON...}"
```

The store knows nothing about the structure of the value. You set it, you get it by key. That's it.

### Redis Internals

Redis is an **in-memory** store with optional persistence. All data lives in RAM — this is why it's fast (sub-millisecond reads/writes).

**Data structures Redis supports** (not just strings):

| Type | Commands | Use case |
|---|---|---|
| String | GET, SET, INCR | Counters, cached values, session tokens |
| Hash | HGET, HSET, HMGET | Object fields (user profile) |
| List | LPUSH, RPOP, LRANGE | Message queues, activity feeds |
| Set | SADD, SMEMBERS, SINTER | Unique tags, friend lists |
| Sorted Set | ZADD, ZRANGE, ZRANK | Leaderboards, rate limiting, time-series |
| Bitmap | SETBIT, BITCOUNT | Presence tracking (daily active users) |
| HyperLogLog | PFADD, PFCOUNT | Approximate cardinality (unique visitors) |
| Stream | XADD, XREAD | Event logs, message queues |

**Persistence options:**

- **RDB (snapshot):** Periodically dump full in-memory state to disk. Fast restarts, potential data loss since last snapshot.
- **AOF (Append-Only File):** Log every write command. Replay on restart. Near-zero data loss, slower restarts.
- **RDB + AOF:** Both — AOF for durability, RDB for faster restarts.

**Redis Cluster:** Consistent hashing across 16384 slots. Keys distributed across nodes. Each node has primary + replica.

### When to Use

- Session storage
- Caching (DB query results, computed values)
- Rate limiting (INCR + EXPIRE)
- Pub/Sub messaging
- Leaderboards (Sorted Sets)
- Distributed locks (Redisson, SET NX PX)

### When NOT to Use

- Persistent primary data store (data fits in RAM constraint)
- Complex queries (no joins, no filters on values)
- Large blobs (Redis is optimised for small values)

---

## 3. Document Stores — MongoDB

### Data Model

Data is stored as **self-describing documents** — typically JSON (internally BSON in MongoDB). Documents in a collection can have different fields.

```json
// users collection
{
  "_id": "u42",
  "name": "Vidhan",
  "email": "v@x.com",
  "addresses": [
    {"type": "home", "city": "Nagpur", "pincode": "440001"},
    {"type": "work", "city": "Mumbai", "pincode": "400001"}
  ],
  "preferences": {
    "notifications": true,
    "theme": "dark"
  }
}
```

No rigid schema. One user has 2 addresses, another has none — both valid. Nested objects and arrays are first-class citizens.

### MongoDB Internals

**Storage engine — WiredTiger:**
Documents stored in B-Tree structures on disk. Each collection is a B-Tree; each index is a separate B-Tree. Supports MVCC at the document level — concurrent reads/writes without collection-level locks.

**Indexing:**
MongoDB supports B-Tree indexes on any field, including nested fields and array elements:

```javascript
db.orders.createIndex({ "user_id": 1, "created_at": -1 })
db.products.createIndex({ "attributes.color": 1 })  // nested field
db.orders.createIndex({ "items.product_id": 1 })    // array element
```

**Aggregation Pipeline:**
Unlike key-value stores, MongoDB supports complex queries:

```javascript
db.orders.aggregate([
  { $match: { status: "DELIVERED" } },
  { $group: { _id: "$user_id", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

**Consistency model:**
MongoDB defaults to eventual consistency on secondaries. For strong consistency, read from primary or use `readConcern: "majority"`.

**Transactions:**
MongoDB 4.0+ supports multi-document ACID transactions — but they're more expensive than single-document operations. Design to avoid them where possible.

### When to Use

- Variable schema data (product catalog with different attributes per category)
- Hierarchical / nested data (orders with embedded line items)
- Rapid iteration (schema evolves without migrations)
- Content management, user profiles, event logs

### When NOT to Use

- Heavy relational data with lots of joins
- Financial data requiring strict ACID (use PostgreSQL)
- Data that fits naturally in tables — don't use a document store just because it's trendy

---

## 4. Column-Family Stores — Cassandra

### Data Model

Often misunderstood. Cassandra is NOT just a wide table. It's a **distributed, partitioned, sorted map**.

Data is organised as:

```
(partition_key) → sorted map of (clustering_key → columns)
```

```
Table: order_events (partition_key: user_id, clustering_key: event_time)

user_id="u42" → [
  {event_time: 2024-01-01 10:00, event_type: "PLACED",   amount: 500},
  {event_time: 2024-01-01 10:05, event_type: "CONFIRMED", amount: 500},
  {event_time: 2024-01-02 14:00, event_type: "SHIPPED",   amount: 500},
]

user_id="u99" → [
  {event_time: 2024-01-03 09:00, event_type: "PLACED",   amount: 200},
]
```

All rows with the same `user_id` are stored together on the same node (same partition) and sorted by `event_time` on disk. This makes "give me all events for user u42, ordered by time" an extremely fast operation — single node, sequential disk read.

### Cassandra Internals

**LSM-Tree storage (not B-Tree):**
Writes go to an in-memory structure (Memtable) and the commit log. Periodically, Memtables are flushed to disk as immutable SSTable files. Reads merge data across SSTables and Memtable. Compaction runs in the background to merge and garbage collect SSTables.

This is why Cassandra is write-optimised — writes are always sequential (in-memory + commit log append), never random.

**No single point of failure:**
Every node in a Cassandra cluster is equal (no primary). Data is replicated using consistent hashing. Any node can serve any request (coordinator routes to the right replica).

**Tunable consistency:**
Per-query consistency level — ONE, QUORUM, ALL. You choose the trade-off per operation.

**What Cassandra cannot do:**
- No joins (data must be denormalized)
- No aggregations (no SUM, COUNT across partitions — must be done in application)
- No ad-hoc queries on non-primary-key columns (must know access pattern upfront to design table)
- No cross-partition transactions

### When to Use

- Time-series data (IoT sensor readings, event logs, metrics)
- Write-heavy workloads (millions of writes/second)
- Data that's always accessed by the same partition key
- Geo-distributed systems (multi-datacenter replication built in)
- High availability requirements (no single point of failure)

### When NOT to Use

- You need joins or ad-hoc queries
- Access patterns are unknown or highly variable
- Low write volume — Cassandra's operational complexity isn't justified

---

## 5. Graph Databases — Neo4j

### Data Model

Data stored as **nodes, relationships, and properties**. No tables, no documents — just entities and the connections between them.

```
(User {id: "u42", name: "Vidhan"})
    -[:PURCHASED]→ (Product {id: "p10", name: "Laptop"})
    -[:REVIEWED {rating: 5}]→ (Product {id: "p10"})
    -[:FOLLOWS]→ (User {id: "u99", name: "Rahul"})

(Product {id: "p10"})
    -[:IN_CATEGORY]→ (Category {name: "Electronics"})
    -[:SIMILAR_TO]→ (Product {id: "p11", name: "Monitor"})
```

### Internals — Index-Free Adjacency

The critical property of graph databases: each node stores **direct pointers to its adjacent nodes**. Traversing a relationship is O(1) — just follow a pointer.

In a relational DB, "find all friends of friends of user 42" requires:
```sql
SELECT f2.user_id FROM friends f1
JOIN friends f2 ON f1.friend_id = f2.user_id
WHERE f1.user_id = 42
```
This requires two table scans with joins. At depth 5 (5-hop traversal), it's 5 nested joins — performance collapses.

In Neo4j, depth-5 traversal is 5 pointer follows per node — performance is proportional to the number of nodes actually visited, not the total graph size.

**Cypher query language:**
```cypher
// Find all products purchased by users who follow Vidhan
MATCH (vidhan:User {id: "u42"})-[:FOLLOWS]->(friend:User)
                                -[:PURCHASED]->(product:Product)
RETURN product.name, COUNT(friend) AS buyers
ORDER BY buyers DESC
LIMIT 10
```

### When to Use

- Social networks (friend recommendations, follower graphs)
- Recommendation engines ("users who bought X also bought Y")
- Fraud detection (detecting ring-shaped transaction patterns)
- Knowledge graphs
- Access control (role hierarchies, permission inheritance)

### When NOT to Use

- Simple entity storage (overkill — use PostgreSQL)
- Write-heavy workloads (Neo4j is read-optimised for traversal)
- Large-scale analytical queries across all nodes

---

## 6. The Full Comparison

| Property | Key-Value (Redis) | Document (MongoDB) | Column-Family (Cassandra) | Graph (Neo4j) |
|---|---|---|---|---|
| Data model | Flat key→value | JSON documents | Partitioned sorted rows | Nodes + edges |
| Query flexibility | Only by key | Rich (indexed fields) | Only by partition key | Deep traversal |
| Write speed | Extremely fast (in-memory) | Fast | Extremely fast (LSM) | Moderate |
| Read speed | Extremely fast | Fast (indexed) | Fast (known partition) | Fast (traversal) |
| Horizontal scale | Redis Cluster | Sharding | Native (built-in) | Limited |
| Consistency | Eventual / configurable | Configurable | Tunable (ONE/QUORUM/ALL) | ACID (single node) |
| Schema | None | Flexible | Fixed partition/clustering keys | Flexible |
| Joins | None | Limited ($lookup) | None | Native (traversal) |
| Best for | Caching, sessions | Flexible content | Time-series, write-heavy | Relationships |

---

## 7. ShopSphere Mapping

```
Service              DB Choice           Why
─────────────────────────────────────────────────────────────
User Service         PostgreSQL          Auth, ACID, FK constraints
Order Service        PostgreSQL          Transactions, FK to users/products
Payment Service      PostgreSQL          ACID, audit trail
Product Service      PostgreSQL +        Core data relational;
                     MongoDB (attrs)     variable attributes per category
Search Service       Elasticsearch       Full-text + faceted search
Cart Service         Redis               Ephemeral, fast, expires naturally
Session Service      Redis               Sub-ms reads, TTL built-in
Notification Svc     MongoDB             Variable payload per channel
Analytics Service    Cassandra           High-write event ingestion
Recommendation Svc   Neo4j               "Users who bought X also bought Y"
Rate Limiting        Redis               INCR + EXPIRE per IP/user
```

**Product attributes in MongoDB — why:**
A laptop has `{RAM: "16GB", GPU: "RTX4080", battery: "72Wh"}`.
A T-shirt has `{size: "M", color: "blue", material: "cotton"}`.
These can't share a PostgreSQL schema cleanly without a horrible EAV (Entity-Attribute-Value) table. MongoDB stores each product as a document with its own shape — clean and queryable.

---

## 8. Interview Q&A

**Q: Why can't you just use PostgreSQL for everything?**
A: PostgreSQL is excellent for structured, relational data with complex queries. But it struggles at scale for: (1) write-heavy time-series (Cassandra's LSM-tree is purpose-built for this), (2) in-memory caching (Redis serves sub-millisecond reads from RAM), (3) graph traversal at depth (PG joins collapse at 4+ hops), (4) flexible schemas with variable attributes (MongoDB handles naturally, PG needs EAV or JSONB with trade-offs). Use PostgreSQL as the default; reach for NoSQL when a specific access pattern justifies the operational cost.

**Q: What is index-free adjacency and why does it matter?**
A: In graph databases like Neo4j, each node stores direct physical pointers to its neighbors. Traversing a relationship is a pointer follow — O(1). In a relational DB, finding neighbors requires a join on a foreign key table — O(log N) at minimum, collapsing at deep traversal depths. Index-free adjacency makes multi-hop traversals fast regardless of graph size; relational joins get exponentially slower with each hop.

**Q: Why is Cassandra write-optimised?**
A: Cassandra uses an LSM-Tree (Log-Structured Merge Tree). Writes go to an in-memory Memtable and a sequential commit log — both are sequential writes, the fastest operation on any storage medium. There's no random in-place update. The B-Tree (used by PostgreSQL/MySQL) requires finding the right page and updating it in place — random I/O. LSM-Tree defers all random I/O to background compaction, making the write path purely sequential and extremely fast.

**Q: When would you choose MongoDB over PostgreSQL for a new service?**
A: When: (1) the schema is genuinely variable across entities (different product types, dynamic form responses), (2) the data is hierarchical and embedding sub-documents avoids expensive joins, (3) the team is iterating fast and schema rigidity would slow development. Not when: you have relational data, need transactions across multiple entities, or are choosing it just because it's "scalable" — PostgreSQL with read replicas handles enormous workloads.

**Q: How does Redis handle data loss if it's all in memory?**
A: Redis has two persistence mechanisms. RDB takes periodic snapshots — fast to restore but risks losing data since the last snapshot. AOF logs every write command — replayed on restart for near-zero data loss but slower startup. Production setups use both: AOF for durability, RDB as a fast-restore fallback. Redis Cluster also replicates each shard to at least one replica, so a single node failure doesn't lose data.

---

Ready for **Topic 8: Polyglot Persistence — using different databases per service based on access patterns**?
