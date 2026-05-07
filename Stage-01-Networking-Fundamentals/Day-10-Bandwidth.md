# Topic 10: Bandwidth

## Theory

Bandwidth is one of those terms that gets casually misused constantly — even by engineers. Let's fix that permanently.

**Bandwidth** is the **maximum amount of data that can be transmitted over a network link per unit of time**. It is the capacity of the pipe — not the speed of water through it, not how fast a single packet travels, not how responsive your API is.

```
Bandwidth  = the number of lanes on a highway
Latency    = time for one car to drive from A to B
Throughput = actual number of cars passing per hour
             (always ≤ bandwidth, usually less)
```

Bandwidth is measured in **bits per second**:
- `Kbps` — kilobits per second (thousands)
- `Mbps` — megabits per second (millions) — home internet
- `Gbps` — gigabits per second (billions) — datacenter links
- `Tbps` — terabits per second (trillions) — backbone internet links, CDN infrastructure

**Critical distinction — bits vs bytes:**
```
1 byte = 8 bits

100 Mbps bandwidth = 100 megabits per second
                   = 12.5 megabytes per second

Always check whether a spec is in bits (Mbps) or bytes (MB/s)
Network specs → bits    (Mbps, Gbps)
Storage specs → bytes   (MB/s, GB/s)
```

---

## Internals — What Bandwidth Actually Represents

Bandwidth is a property of the **physical link** between two points:

```
Your laptop  ──(WiFi 300Mbps)──  Router  ──(1Gbps fibre)──  ISP  ──(10Gbps backbone)──  Server
```

The **bottleneck** is always the narrowest link in the chain — the weakest link determines the actual throughput. Even if your server has a 10Gbps NIC and your ISP has a 10Gbps backbone, if your home WiFi is 300Mbps, you can never exceed 300Mbps to that server.

**Bandwidth is shared:**
- On a 1Gbps datacenter link, if 10 services are all transferring data simultaneously, they share that 1Gbps
- This is why network I/O intensive workloads on shared cloud instances can be unpredictable
- AWS EC2 instances have a bandwidth cap per instance type — a `t3.micro` gets ~64Mbps, a `c5.18xlarge` gets 25Gbps

**Bandwidth vs actual throughput in practice:**
```
Theoretical bandwidth:    1 Gbps
Protocol overhead (TCP):  ~5%
TLS encryption overhead:  ~3%
Retransmissions:          ~2% (on a good day)
Actual usable throughput: ~900 Mbps
```

Real throughput is always below the theoretical bandwidth ceiling due to protocol overhead, errors, and congestion.

---

## Bandwidth in Different Contexts

**Network bandwidth — between machines:**
```
Within same datacenter rack:    10–100 Gbps (microsecond latency)
Between datacenters (same region): 10 Gbps (low latency)
Cross-continental links:         1–10 Gbps (high latency)
Home broadband:                  100 Mbps – 1 Gbps (variable latency)
4G mobile:                       10–50 Mbps (variable)
5G mobile:                       100 Mbps – 1 Gbps (low latency)
```

**Disk bandwidth — reading/writing data:**
```
HDD sequential read/write:    100–200 MB/s
SSD (SATA) sequential:        500 MB/s
NVMe SSD sequential:          3,000–7,000 MB/s
In-memory (RAM) read:         ~50,000 MB/s
```

This is why databases use memory-mapped files and buffer pools — RAM bandwidth is orders of magnitude higher than disk bandwidth.

**Why disk bandwidth matters for databases:**
- PostgreSQL doing a full table scan on a 100GB table over an HDD = ~500 seconds minimum
- Same scan on NVMe SSD = ~14 seconds
- Same data in memory = under 2 seconds
- This is the fundamental reason why proper indexing, caching, and memory sizing matter so much

---

## Bandwidth-Bound vs Latency-Bound Systems

Not all systems are constrained by the same thing. Identifying the constraint is the first step to fixing it.

**Bandwidth-bound workloads:**
- Large file transfers — video uploads, backups, bulk exports
- DB replication — streaming WAL logs from primary to replicas
- Bulk Elasticsearch indexing
- Streaming analytics pipelines ingesting high-volume events
- CDN origin-to-edge data distribution

*Symptom:* High data volume, CPU and memory are fine, but the pipeline is slow because the pipe is full.

**Latency-bound workloads:**
- Interactive APIs — user is waiting for a response
- Payment processing — must complete quickly for good UX
- Real-time search autocomplete — must respond in < 100ms
- WebSocket message delivery

*Symptom:* Data volume is small per operation, but the round-trip time dominates.

**A common mistake** — throwing more bandwidth at a latency-bound problem. If your API is slow because of a slow DB query, more bandwidth does nothing. If your file upload is slow because of bandwidth saturation, caching and query optimisation do nothing.

---

## Bandwidth Cost and Estimation — Critical for System Design Interviews

In system design interviews you are expected to estimate bandwidth requirements. Here is the framework:

**Formula:**
```
Bandwidth required = Request rate × Average payload size

Convert to bits:
1 KB = 8,000 bits
1 MB = 8,000,000 bits
1 GB = 8,000,000,000 bits
```

**Example — estimating ShopSphere bandwidth:**

```
Scenario: ShopSphere handles 10,000 requests/second at peak

API responses:
  Average JSON response = 5 KB
  10,000 req/s × 5 KB × 8 = 400 Mbps outbound bandwidth needed

Image serving (via CDN):
  50,000 image requests/second (CDN handles most)
  Average image = 200 KB
  50,000 × 200 KB × 8 = 80 Gbps ← this is why you need a CDN
  CDN serves ~95% from edge = 4 Gbps from origin

Video (if applicable):
  1080p video stream = ~8 Mbps per user
  10,000 concurrent viewers × 8 Mbps = 80 Gbps ← CDN is non-negotiable
```

This kind of back-of-envelope calculation directly informs architecture decisions — whether you need a CDN, how many servers you need, what instance types to choose.

---

## Bandwidth Optimisation Techniques

**1. Compression:**
```
Uncompressed JSON response:  50 KB
Gzip compressed:             8 KB   (84% reduction)
Brotli compressed:           6 KB   (88% reduction)

HTTP header to enable:
Accept-Encoding: gzip, br          (client signals support)
Content-Encoding: gzip             (server confirms compression used)
```
Enable compression at your Nginx/API Gateway layer — single config change, dramatic bandwidth reduction.

**2. CDN — push bandwidth-intensive content to the edge:**
```
Without CDN:
  10,000 users in India download images from US server
  Each image = 200KB, 10,000 × 200KB = 2GB across the Atlantic

With CDN (Cloudflare/CloudFront):
  Images cached at Mumbai edge node
  0 bytes cross the Atlantic after first cache warm
  Latency: 150ms → 10ms
```

**3. Efficient serialisation formats:**
```
JSON:         {"userId":"u-123","productId":"p-456","qty":2}   = 49 bytes
Protobuf:     binary encoding of same data                      = 12 bytes  (75% smaller)
MessagePack:  binary JSON                                       = 18 bytes  (63% smaller)
```
gRPC uses Protobuf — this is one reason internal service communication over gRPC is more bandwidth-efficient than REST+JSON.

**4. Pagination and field filtering:**
```
Bad:  GET /api/products → returns all 10,000 products, 50MB response
Good: GET /api/products?page=1&size=20&fields=id,name,price → 20 products, 2KB response
```

**5. Caching — eliminate bandwidth entirely:**
```
Cache hit = 0 bytes of bandwidth to origin server
Cache miss = full response bandwidth cost

CDN cache hit ratio of 90% means 90% of bandwidth never leaves the edge
```

---

## Real-World Example — ShopSphere Bandwidth Planning

**Problem:** ShopSphere product images are served directly from the Spring Boot app. Users in India are experiencing slow image loads because the server is in Singapore.

**Analysis:**
```
Current setup:
  Product images average: 800KB (unoptimised)
  Peak image requests: 5,000/second
  Bandwidth: 5,000 × 800KB × 8 = 32 Gbps ← impossible without massive infrastructure

After image optimisation (WebP format, resizing):
  Images average: 150KB
  Bandwidth: 5,000 × 150KB × 8 = 6 Gbps ← still too much for app servers

After CDN (CloudFront with India edge):
  95% served from edge: 5% × 6 Gbps = 300 Mbps from origin ← manageable
  Latency for Indian users: 150ms → 15ms ← massive UX improvement
```

**Additional bandwidth optimisations applied:**
- Enable Gzip on all JSON API responses — reduces API bandwidth by ~85%
- Return paginated product listings — 20 items instead of full catalogue
- Use Protobuf for internal service-to-service communication on high-frequency paths
- Set proper `Cache-Control` headers so browsers cache static assets locally

---

## Interview Q&A

**Q: What is bandwidth and how is it different from latency and throughput?**
Bandwidth is the maximum capacity of a network link — how much data can flow per second at most. Latency is the time a single operation takes end to end. Throughput is the actual amount of data or requests processed per second, which is always less than or equal to bandwidth. A wide pipe does not mean fast delivery — high bandwidth with high latency still means slow interactive response times.

**Q: Why is bandwidth a concern in system design even with fast modern networks?**
Because bandwidth costs money, has hard limits per server/instance, and becomes the bottleneck for data-intensive workloads. A single 1080p video stream requires ~8Mbps — with 100,000 concurrent viewers that is 800Gbps, which is only feasible with a CDN distributing the load across hundreds of edge nodes. Without bandwidth planning, a viral moment can bankrupt or collapse a system.

**Q: How would you reduce bandwidth usage in a high-traffic API?**
Enable response compression (Gzip or Brotli) at the reverse proxy layer, implement pagination to limit response sizes, use field filtering so clients only receive the data they need, cache responses at the CDN and browser level to eliminate repeat bandwidth costs, and use binary serialisation formats like Protobuf for internal service communication where JSON verbosity is unnecessary.

**Q: What is the relationship between bandwidth and database performance?**
Disk bandwidth determines how fast a database can read data from storage — a full table scan on an HDD is orders of magnitude slower than on NVMe SSD, which is still far slower than reading from memory. This is why proper indexing eliminates sequential scans, why databases size their buffer pool to keep hot data in RAM, and why read replicas must have sufficient network bandwidth to keep up with the primary's WAL stream without replication lag growing unbounded.

**Q: How do you estimate bandwidth requirements in a system design interview?**
Multiply request rate by average payload size to get bits per second. For example, 10,000 requests per second with an average response of 5KB equals 400Mbps of outbound bandwidth. For media-heavy systems, account for image and video traffic separately — a 200KB image served 50,000 times per second requires 80Gbps, which immediately tells you a CDN is non-negotiable. Always separate origin bandwidth from edge bandwidth after CDN caching.

---

Say **"next"** when ready for the final topic of Stage 1 — Topic 11: Communication Protocols (REST, gRPC, GraphQL).
