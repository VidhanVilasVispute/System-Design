# Complete System Design Roadmap — Beginner to Advanced

---

## Stage 1 — Networking Fundamentals

- **Client-Server Model** — who initiates requests, who responds, what a server actually is
- **DNS Resolution** — what happens when you type a URL, recursive vs iterative resolution
- **IP Addresses & Ports** — IPv4 vs IPv6, how machines locate each other, port ranges
- **TCP vs UDP** — TCP's reliability (handshake, ACK, retransmission) vs UDP's speed, when to use each
- **HTTP / HTTPS** — request/response cycle, methods (GET, POST, PUT, DELETE), status codes, headers
- **HTTP/1.1 vs HTTP/2 vs HTTP/3** — multiplexing, head-of-line blocking, QUIC protocol
- **WebSockets** — persistent bidirectional connection, when to use over HTTP (chat, live feeds)
- **Long Polling vs SSE** — alternatives to WebSocket for one-directional real-time data
- **Latency vs Throughput** — two completely different things, how to measure, how to optimize each
- **Bandwidth** — the pipe capacity, not the speed of data through it
- **Communication Protocols** — REST, gRPC, GraphQL — trade-offs, when to use which

---

## Stage 2 — APIs & Communication Patterns

- **What is an API** — contract between services, why it matters in distributed systems
- **REST API Design** — resource naming, idempotency, versioning, pagination, error responses
- **gRPC** — protocol buffers, strongly typed contracts, better for internal service-to-service calls
- **GraphQL** — client-driven queries, solves over-fetching and under-fetching
- **Synchronous Communication** — caller waits for response, tight coupling, simpler but fragile
- **Asynchronous Communication** — fire and forget, decoupled, resilient but harder to debug
- **Message-Based Communication** — services talk through a broker, not directly to each other
- **Publisher-Subscriber Model** — publishers emit events, subscribers react independently, loose coupling
- **Event-Driven Architecture** — system state changes drive behaviour, highly decoupled design
- **Authentication vs Authorization** — AuthN is who you are, AuthZ is what you can do
- **JWT & OAuth2** — how stateless auth works, token structure, access vs refresh tokens
- **API Gateway** — single entry point, handles auth, rate limiting, routing, and SSL termination

---

## Stage 3 — Core Building Blocks

- **Load Balancers** — L4 (TCP level) vs L7 (HTTP-aware), routing algorithms (round robin, least connections, IP hash)
- **Forward Proxy vs Reverse Proxy** — forward hides the client, reverse hides the server, Nginx as reverse proxy
- **CDN** — geographically distributed edge nodes, push vs pull CDN, cache invalidation
- **Caching** — why it exists, where to place it (client, CDN, server, DB), Redis vs Memcached
- **Cache Eviction Policies** — LRU, LFU, MRU, FIFO, LIFO, Random Replacement — trade-offs of each
- **Cache Strategies** — read-through, write-through, write-behind, cache-aside — who populates the cache
- **Cache Invalidation** — the hardest problem in caching, TTL, event-based invalidation
- **Message Queues** — Kafka vs RabbitMQ, when to use each, dead letter queues, consumer groups
- **SQL vs NoSQL** — relational for structure and transactions, NoSQL for scale and flexibility
- **Object Storage** — S3-style blob storage for files, images, videos — not a filesystem
- **Different Storage Systems** — block storage, file storage, object storage, in-memory — know the difference

---

## Stage 4 — Databases Deep Dive

- **RDBMS Internals** — how indexes work (B-tree), query planning, ACID guarantees
- **Indexing** — primary vs secondary index, composite index, covered index, cost of over-indexing
- **RDBMS Horizontal Scaling** — why it is hard, read replicas, connection pooling, partitioning
- **Database Read Replicas** — offload read traffic, eventual consistency trade-off
- **Database Sharding** — horizontal partitioning by shard key, hot shard problem, resharding cost
- **Consistent Hashing** — assign data to nodes, minimal reshuffling when nodes join/leave
- **NoSQL Categories** — key-value (Redis), document (MongoDB), column-family (Cassandra), graph (Neo4j)
- **Polyglot Persistence** — use different databases for different services based on access patterns
- **Connection Pooling** — reuse DB connections, why creating a new connection per request kills you
- **ACID vs BASE** — strong consistency vs eventual consistency, the real trade-off behind SQL vs NoSQL
- **Schema Evolution** — adding/removing columns safely, backward compatibility, rolling migrations
- **LSM-Tree vs B-Tree** — why Cassandra favours writes, why MySQL favours reads — the internals

---

## Stage 5 — Scalability

- **Horizontal vs Vertical Scaling** — more machines vs bigger machine, why horizontal wins long-term
- **Stateless Services** — no session on the server, a prerequisite for horizontal scaling
- **Rate Limiting** — token bucket vs sliding window vs fixed window, distributed rate limiting
- **Single Point of Failure (SPOF)** — identifying SPOFs, eliminating them with redundancy
- **Redundancy and Replication** — active-active vs active-passive, what gets replicated and why
- **Thundering Herd Problem** — what happens when cache expires and 10,000 requests hit the DB at once
- **Back-pressure** — how a downstream service signals it is overwhelmed to slow down the upstream
- **Autoscaling** — horizontal pod autoscaling, scaling triggers, cooldown periods

---

## Stage 6 — Reliability & Availability

- **CAP Theorem** — Consistency, Availability, Partition Tolerance — you only get two, pick wisely
- **Availability Numbers** — 99.9% vs 99.99% vs 99.999%, what downtime that actually means per year
- **Replication** — synchronous (stronger consistency, slower) vs asynchronous (faster, risk of data loss)
- **Failover** — automatic promotion of replica when primary goes down, failover detection lag
- **Circuit Breakers** — stop calling a failing service, fail fast, give it time to recover
- **Retry Logic with Exponential Backoff** — retry intelligently with jitter, don't hammer a struggling service
- **Bulkhead Pattern** — isolate thread pools per dependency so one slow service doesn't kill everything
- **Timeout & Deadline Propagation** — set timeouts at every layer, propagate deadlines across services
- **SLA / SLO / SLI** — how uptime is contractually defined, measured, and monitored in production
- **Health Checks** — liveness vs readiness probes, how load balancers know a node is alive

---

## Stage 7 — Distributed Systems

- **Why Distributed Systems Are Hard** — partial failures, network unreliability, no global clock
- **Consistency Models** — strong, eventual, causal, read-your-writes — which systems offer what
- **Lamport Logical Clocks** — assign ordering to events across nodes without a shared clock
- **Vector Clocks** — track causality across nodes, detect concurrent conflicting writes
- **Consensus Algorithms (Raft, Paxos)** — how nodes agree on a value when some may fail
- **Leader Election** — picking one coordinator node without a central authority
- **Quorum Reads/Writes** — how Cassandra and DynamoDB balance consistency vs availability
- **Two-Phase Commit (2PC)** — distributed transactions, why it blocks and why most avoid it
- **Saga Pattern** — alternative to 2PC, choreography vs orchestration, compensating transactions
- **Event Sourcing** — store the log of changes, not just current state, replay to reconstruct
- **CQRS** — separate read and write models, why it helps in high-read or complex domains
- **Outbox Pattern** — reliably publish events even if your service crashes mid-write
- **Write-Ahead Log (WAL)** — how databases guarantee durability before confirming writes
- **The Power of the Log** — the log as the source of truth, how Kafka and databases both rely on it

---

## Stage 8 — Microservices & Architecture Patterns

- **Microservice Architecture** — advantages (independent deploy, scale, tech), disadvantages (operational complexity)
- **Service Discovery** — how services find each other dynamically (Consul, Eureka, Kubernetes DNS)
- **Sidecar Pattern** — offload logging, auth, observability to a co-located process (Envoy, Istio)
- **API Composition** — aggregate responses from multiple services for a single client request
- **Data Ownership** — each service owns its own DB, no shared databases between services
- **Distributed Tracing** — trace a request across 10 services, correlation IDs, Jaeger, Zipkin
- **Service Mesh** — Istio/Linkerd handle retries, mTLS, observability at the infrastructure layer
- **Blue-Green Deployments** — two production environments, flip traffic with zero downtime
- **Canary Deployments** — roll out to 1% first, watch metrics, then gradually expand
- **Feature Flags** — decouple deploy from release, roll out features independently of code deploys

---

## Stage 9 — Data-Intensive & Advanced Storage

- **OLTP vs OLAP** — transactional databases vs analytical warehouses, never mix workloads
- **Column-Oriented Storage** — why Parquet and BigQuery are faster for analytics (read fewer columns)
- **Geospatial Data** — location databases, quadtrees, Hilbert curves, PostGIS, geohashing
- **Full-Text Search** — inverted index, relevance scoring, Elasticsearch internals
- **Time-Series Databases** — InfluxDB, Prometheus — optimised for append-heavy, time-ordered data
- **Batch vs Stream Processing** — when each approach fits, latency vs throughput trade-off
- **Stream Processing** — Flink, Spark Streaming for real-time pipelines
- **MapReduce** — the foundational paradigm behind large-scale batch processing

---

## Stage 10 — Classic System Design Problems

- **URL Shortener** — hashing, collision handling, redirects, analytics, scale
- **Rate Limiter** — token bucket vs sliding window, distributed counters
- **Notification System** — push, email, SMS, priority queuing, fan-out
- **Search Autocomplete** — trie vs inverted index, latency requirements
- **Twitter / Feed** — fan-out on write vs read, timeline generation, the celebrity problem
- **WhatsApp / Chat** — message delivery guarantees, ordering, end-to-end encryption model
- **Instagram** — photo upload pipeline, feed generation, CDN delivery
- **YouTube / Netflix** — video upload, transcoding pipeline, adaptive bitrate, CDN
- **Tinder** — proximity search, geohashing, swipe matching, recommendation feed
- **Uber / Location Service** — quadtrees, real-time location updates, driver matching
- **Google Drive** — chunked upload, versioning, conflict resolution, sync protocol

---

**Recommended Books alongside this roadmap:**
- *Designing Data-Intensive Applications* — Martin Kleppmann (essential for Stages 5–9)
- *System Design Interview Vol 1 & 2* — Alex Xu (essential for Stage 10)
