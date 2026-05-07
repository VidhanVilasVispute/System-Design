# Stage 3 — Topic 4: Caching

## Theory

Caching is the single most impactful performance optimisation in distributed systems. Almost every major system design answer — URL shortener, Twitter, YouTube, Uber — involves caching as a core component. The difference between a junior and senior answer is not whether you mention caching, but **where you put it, what strategy you use, what you evict, and what happens when the cache fails**.

**The fundamental insight:**
```
Reading the same data from a DB repeatedly is wasteful.
Most data is read far more than it is written.
Store frequently read data in a faster layer closer to the reader.

Without cache:
  Every product page request → DB query → 10ms
  1,000,000 page views/day → 1,000,000 DB queries/day

With cache:
  First request → DB query → store in Redis → 10ms
  Next 999,999 requests → Redis lookup → 1ms each
  DB receives 1 query instead of 1,000,000
```

**The caching hierarchy — speed vs capacity:**
```
L1 CPU Cache:     ~0.5ns  — bytes, built into processor
L2 CPU Cache:     ~7ns    — kilobytes
L3 CPU Cache:     ~30ns   — megabytes
RAM:              ~100ns  — gigabytes
Redis (local):    ~0.1ms  — gigabytes, network hop
Redis (remote):   ~1ms    — terabytes, datacenter
SSD:              ~0.1ms  — terabytes
HDD:              ~10ms   — petabytes
Network (remote): ~30ms+  — unlimited
```

Each layer is orders of magnitude slower than the one above it. Caching moves data up the hierarchy — closer to the reader, faster to retrieve.

---

## Where Caches Live — The Four Positions

### Position 1 — Client-Side Cache

```
Browser / Mobile App
  ↓
[Browser Cache / App Memory Cache]
  ↓  (only on cache miss)
CDN Edge
  ↓  (only on cache miss)
Origin Server

Browser caches:
  HTTP responses with Cache-Control headers
  Service Worker cache (PWA — works offline)
  localStorage / sessionStorage (application data)

Mobile app caches:
  In-memory LRU cache for API responses
  SQLite for offline data
  Image cache (Glide, Picasso on Android)
  
Latency: 0ms (in-memory hit) to 5ms (disk cache hit)
Scope: single user, single device
```

### Position 2 — CDN Cache

```
Already covered in Topic 3.
Sits between client and origin.
Caches public, static, or semi-static content.
Serves millions of users from edge nodes globally.
```

### Position 3 — Server-Side Application Cache

```
API Gateway / Microservice
  ↓
[Redis / Memcached]
  ↓  (only on cache miss)
Database

Sits in front of the database within your infrastructure.
Shared across all service instances.
Reduces DB load, improves response time.

This is what most people mean by "adding a cache."
```

### Position 4 — Database Cache

```
Database has its own internal caching:
  PostgreSQL buffer pool — frequently read pages in RAM
  MySQL InnoDB buffer pool — same
  Elasticsearch field data cache

These are automatic and internal — you tune them by
allocating more RAM to the database process.
Not the same as an external cache like Redis.
```

---

## Redis vs Memcached — The Decision

Both are in-memory key-value stores. In practice Redis dominates — you should know why:

| Feature | Redis | Memcached |
|---|---|---|
| Data structures | Strings, hashes, lists, sets, sorted sets, streams, geospatial | Strings only |
| Persistence | RDB snapshots + AOF write-ahead log | None — purely in-memory |
| Replication | Built-in primary-replica replication | Not built-in |
| Clustering | Redis Cluster — automatic sharding | Client-side sharding only |
| Pub/Sub | Built-in | None |
| Lua scripting | Yes — atomic custom operations | No |
| Transactions | MULTI/EXEC | None |
| TTL | Per-key TTL | Per-key TTL |
| Memory efficiency | Slightly less efficient | Slightly more efficient |
| Multithreading | Single-threaded (I/O multi-threaded in Redis 6+) | Multi-threaded |

**When Memcached makes sense:**
```
Pure simple key-value caching only
Need maximum memory efficiency for simple string values
Multi-threaded performance for very high throughput simple GET/SET
No need for persistence, replication, or advanced data structures
```

**Redis for everything else** — its versatility makes it the default choice. In ShopSphere, Redis serves as:
- Application cache (product data, session data)
- Rate limiting counters
- Pub/Sub backplane for WebSocket scaling
- Distributed lock manager
- Leaderboard (sorted sets)
- Session store

---

## Cache Strategies — The Core Four

This is the most important section. Each strategy answers two questions: **who populates the cache** and **when**?

### Strategy 1 — Cache-Aside (Lazy Loading)

The most common strategy. The application manages the cache explicitly.

```
READ flow:
  Application checks cache for key
    HIT  → return cached value
    MISS → query database
           store result in cache with TTL
           return result

WRITE flow:
  Application writes to database
  Invalidates (deletes) the cache key
  Next read will be a cache miss → repopulates from DB
```

```java
// ShopSphere — Cache-Aside for product data
@Service
public class ProductService {

    @Autowired private ProductRepository productRepository;
    @Autowired private RedisTemplate<String, Product> redisTemplate;

    private static final Duration PRODUCT_TTL = Duration.ofMinutes(30);
    private static final String CACHE_PREFIX = "product:";

    public Product getProduct(String productId) {
        String cacheKey = CACHE_PREFIX + productId;

        // 1. Check cache
        Product cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;  // cache HIT
        }

        // 2. Cache MISS — query database
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // 3. Store in cache with TTL
        redisTemplate.opsForValue().set(cacheKey, product, PRODUCT_TTL);

        return product;
    }

    public Product updateProduct(String productId, UpdateProductRequest request) {
        // 1. Update database (source of truth)
        Product updated = productRepository.save(buildUpdated(productId, request));

        // 2. Invalidate cache — next read will fetch fresh from DB
        redisTemplate.delete(CACHE_PREFIX + productId);

        return updated;
    }
}

// Spring annotation equivalent — cleaner
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        return productRepository.findById(productId).orElseThrow(...);
    }

    @CacheEvict(value = "products", key = "#productId")
    public Product updateProduct(String productId, UpdateProductRequest req) {
        return productRepository.save(buildUpdated(productId, req));
    }
}
```

**Cache-Aside pros and cons:**
```
Pros:
  Only requested data gets cached — no wasted memory
  Cache failure is non-fatal — app falls through to DB
  Works with any database
  Simple to understand and implement

Cons:
  Cache miss on first request — cold start latency spike
  Window of inconsistency after write invalidation
  Race condition: two threads miss simultaneously,
                  both query DB, both write to cache
                  (solved with distributed lock or TTL tolerance)
  Developer must explicitly manage cache everywhere
```

### Strategy 2 — Read-Through

The cache sits in front of the database and handles fetching automatically. The application only talks to the cache.

```
READ flow:
  Application requests data from cache
    HIT  → cache returns value
    MISS → cache fetches from database automatically
           cache stores result
           cache returns to application
  Application never directly queries DB

Cache library handles the DB fetch — not application code
```

```java
// Read-through with Spring Cache + custom CacheLoader
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(ProductRepository productRepository) {
        CaffeineCacheManager manager = new CaffeineCacheManager("products");
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .recordStats()
            .build(key -> productRepository.findById((String) key).orElse(null))
            // ↑ CacheLoader — called automatically on cache miss
        );
        return manager;
    }
}

// Application just calls cache — never knows if hit or miss
public Product getProduct(String productId) {
    return cacheManager.getCache("products").get(productId, Product.class);
    // Cache handles DB fetch automatically on miss
}
```

**Read-through pros and cons:**
```
Pros:
  Application code is cleaner — no miss handling logic
  Consistent — cache always contains data or fetches it
  
Cons:
  Cache must know how to talk to the database — tighter coupling
  First request still has miss latency
  Less control over what gets cached and when
```

### Strategy 3 — Write-Through

Every write goes to the cache AND the database simultaneously — both are always in sync.

```
WRITE flow:
  Application writes to cache
  Cache synchronously writes to database
  Both updated atomically — always consistent

READ flow:
  Application reads from cache
  Always a hit (if data was ever written)
  No DB reads needed
```

```java
// Write-through — update both cache and DB on every write
public Product updateProduct(String productId, UpdateProductRequest request) {
    Product updated = buildUpdated(productId, request);

    // 1. Update database
    productRepository.save(updated);

    // 2. Update cache immediately — stays fresh
    redisTemplate.opsForValue().set(
        CACHE_PREFIX + productId,
        updated,
        PRODUCT_TTL
    );

    return updated;
}
```

**Write-through pros and cons:**
```
Pros:
  Cache always consistent with DB
  Read performance is excellent — always cached after first write
  No inconsistency window

Cons:
  Write latency is higher — must write to both
  Cache fills with data that may never be read (write-heavy workloads)
  Cache miss still possible for data never written through
  
Best for: read-heavy workloads where consistency matters
          user session data, user preferences, rate limit counters
```

### Strategy 4 — Write-Behind (Write-Back)

Write to cache immediately. Write to database asynchronously later.

```
WRITE flow:
  Application writes to cache → returns immediately (fast!)
  Background worker asynchronously flushes to database
  
READ flow:
  Read from cache — always fast
  
If cache crashes before flush:
  Data loss — writes in cache not yet in DB are gone
```

```java
// Write-behind with async DB flush
@Service
public class CartService {

    @Autowired private RedisTemplate<String, Cart> redisTemplate;

    public void addToCart(String userId, CartItem item) {
        String cacheKey = "cart:" + userId;

        // 1. Update cache immediately — instant response to user
        Cart cart = redisTemplate.opsForValue().get(cacheKey);
        cart.addItem(item);
        redisTemplate.opsForValue().set(cacheKey, cart, Duration.ofDays(7));

        // 2. Publish event for async DB persistence
        cartEventPublisher.publish(CartUpdatedEvent.of(userId, cart));
        // DB will be updated eventually — user already has response
    }

    // Background consumer flushes cart changes to DB
    @KafkaListener(topics = "cart-events")
    public void persistCart(CartUpdatedEvent event) {
        cartRepository.save(event.getUserId(), event.getCart());
    }
}
```

**Write-behind pros and cons:**
```
Pros:
  Lowest write latency — cache write is synchronous, DB write is async
  DB absorbs write spikes — batch multiple writes into fewer DB operations
  Excellent for write-heavy workloads (shopping cart, view counters)

Cons:
  Data loss risk if cache crashes before DB flush
  Complexity — must handle failures in async flush
  Consistency window — DB may lag behind cache

Best for: non-critical fast-changing data
          shopping carts, view counters, real-time leaderboards,
          anything where brief data loss is acceptable
```

---

## Cache Eviction Policies — Full Coverage

When the cache is full, which items do you evict to make room for new ones?

### LRU — Least Recently Used

```
Evict the item that was accessed least recently.

Cache state (most recent → least recent):
  [product:p-456] [product:p-789] [product:p-123] [product:p-001]

New item arrives, cache full:
  Evict [product:p-001] (least recently used)
  Insert new item at front

Intuition:
  If you haven't used it recently, you probably won't soon.
  Works well for most access patterns with temporal locality.

Used by: Redis (allkeys-lru, volatile-lru), most in-memory caches
Best for: General-purpose caching — the default choice
```

### LFU — Least Frequently Used

```
Evict the item with the lowest access count.

Cache state:
  product:p-456 → accessed 10,000 times  (popular product)
  product:p-789 → accessed 5,000 times
  product:p-123 → accessed 3 times       (unpopular)
  product:p-001 → accessed 1 time        ← evict this

New item arrives, cache full:
  Evict [product:p-001] (lowest frequency)

Intuition:
  Popular items stay regardless of recency.
  Better than LRU for skewed access patterns — top 10% of products
  get 90% of traffic.

Problem: frequency counts are permanent — a product that was
         popular years ago but trending down stays cached forever.
Solution: Use frequency decay — counts decay over time.

Used by: Redis (allkeys-lfu, volatile-lfu since Redis 4.0)
Best for: Skewed workloads with clear popular/unpopular divide
```

### MRU — Most Recently Used

```
Evict the item that was accessed most recently.

Counter-intuitive — why evict what you just accessed?

Use case: Scan patterns where each item is accessed exactly once:
  Sequential file scan — once you finish reading a block, 
  you never need it again.
  
  Stack-based access — last accessed = last needed.
  
Evicting LRU would be wrong here — you'd keep items you'll never use.
MRU recognises: if just accessed, probably done with it.

Rare in application caching — mostly relevant for DB buffer management.
```

### FIFO — First In, First Out

```
Evict the item that was inserted first (oldest).

Cache state (insertion order):
  [p-001 inserted 10min ago] → [p-456 inserted 5min ago] → [p-789 inserted 1min ago]

New item arrives, cache full:
  Evict [p-001] — oldest insertion

Simple but ignores access patterns entirely.
A popular item inserted early gets evicted even if used constantly.

Used for: Simple time-based expiry where insertion order = staleness
          Message queues, event logs
Not suitable for: General-purpose application caching
```

### LIFO — Last In, First Out

```
Evict the most recently inserted item.

Cache state (insertion order):
  [p-001] [p-456] [p-789 ← newest]

New item arrives:
  Evict [p-789] — most recently inserted

Use case: When newest data is least valuable.
  Real-time stream processing — old confirmed data more reliable.
  Stack unwinding — discard recent uncommitted operations.

Very rare in application caching — mostly a theoretical concept
you need to know for interviews.
```

### Random Replacement

```
Evict a randomly chosen item.

No tracking overhead — no recency, no frequency counters.
Surprisingly competitive with LRU for random access patterns.
Used in CPU TLB (Translation Lookaside Buffer) hardware caches.

In software: rarely used intentionally, but Redis's LRU and LFU
are actually approximations using random sampling — not true LRU/LFU.
True LRU requires a doubly-linked list + hash map (O(1) operations).
Redis approximates by sampling N random keys and evicting the best candidate.
```

### Redis Eviction Policies — The Full Set

```
Redis has 8 eviction policies:

noeviction:      Return error when memory full — never evict
                 Dangerous — application sees errors

allkeys-lru:     LRU across ALL keys
volatile-lru:    LRU only among keys WITH expiry set
allkeys-lfu:     LFU across ALL keys  
volatile-lfu:    LFU only among keys WITH expiry set
allkeys-random:  Random among ALL keys
volatile-random: Random among keys WITH expiry
volatile-ttl:    Evict key with nearest expiry first

ShopSphere Redis config:
  maxmemory 4gb
  maxmemory-policy allkeys-lru
  
  "When memory is full, evict the least recently used key
   regardless of whether it has a TTL."
```

---

## The Thundering Herd Problem

A critical cache failure mode that every senior engineer must know:

```
Scenario:
  Product p-456 is a popular item — 10,000 requests/second
  Redis entry expires at exactly 12:00:00.000

At 12:00:00.001:
  All 10,000 requests/second simultaneously see a cache MISS
  All 10,000 simultaneously query the database
  DB collapses under 10,000 concurrent queries
  Each query takes 500ms instead of 2ms
  System appears down
  
This is the Thundering Herd (or Cache Stampede) problem.
```

**Solution 1 — Mutex / Distributed Lock:**
```java
public Product getProduct(String productId) {
    String cacheKey = "product:" + productId;
    String lockKey  = "lock:product:" + productId;

    // Check cache
    Product cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) return cached;

    // Cache miss — try to acquire lock
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

    if (Boolean.TRUE.equals(locked)) {
        // We hold the lock — we fetch from DB
        try {
            Product product = productRepository.findById(productId).orElseThrow();
            redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30));
            return product;
        } finally {
            redisTemplate.delete(lockKey); // release lock
        }
    } else {
        // Another thread is fetching — wait briefly and retry
        Thread.sleep(50);
        return getProduct(productId); // retry — will likely hit cache now
    }
}
```

**Solution 2 — Probabilistic Early Expiry (XFetch):**
```java
// Recompute cache slightly before it expires
// Probabilistically — not all at once

public Product getProduct(String productId) {
    String cacheKey = "product:" + productId;
    CachedProduct cached = redisTemplate.opsForValue().get(cacheKey);

    if (cached != null) {
        double remainingTTL = redisTemplate.getExpire(cacheKey, TimeUnit.SECONDS);
        double delta = 0.1; // tuning parameter
        double recomputeTime = cached.computeTime;

        // XFetch formula — recompute probability increases as TTL decreases
        if (-delta * recomputeTime * Math.log(Math.random()) > remainingTTL) {
            // Probabilistically refresh before expiry
            return refreshCache(productId, cacheKey);
        }
        return cached.product;
    }

    return refreshCache(productId, cacheKey);
}
```

**Solution 3 — Background Refresh:**
```java
// Never let the cache miss — refresh before expiry in background
@Scheduled(fixedDelay = 25 * 60 * 1000)  // every 25 minutes
public void refreshPopularProducts() {
    List<String> popularProductIds = analyticsService.getTopProducts(100);
    for (String productId : popularProductIds) {
        Product product = productRepository.findById(productId).orElseThrow();
        redisTemplate.opsForValue().set(
            "product:" + productId, product, Duration.ofMinutes(30));
    }
    // Cache always warm for top 100 products — thundering herd impossible
}
```

---

## Cache Consistency Patterns

### Write Invalidate vs Write Update

```
Write Invalidate (Cache-Aside default):
  DB updated → cache key deleted
  Next read → cache miss → fresh fetch
  
  Pros: Simple, always consistent on next read
  Cons: One guaranteed cache miss after every write
        Thundering herd if many concurrent readers

Write Update (Write-Through):
  DB updated → cache key updated with new value
  Next read → cache hit → fresh value immediately
  
  Pros: No cache miss after write
  Cons: Wasted work if updated key is never read again
        Complex if update requires DB-generated values (sequences, timestamps)
```

### Double-Delete Pattern — Solving Race Conditions

```
Race condition with simple invalidation:

Thread 1: UPDATE product SET price=99                    t=0
Thread 2: Read cache → MISS → query DB gets old price=89 t=1
Thread 1: Delete cache key                               t=2
Thread 2: Write old price=89 to cache                   t=3

Cache now has stale value that Thread 1 just deleted!

Solution — Double Delete:
Thread 1: Delete cache key                    t=0
Thread 1: UPDATE product SET price=99         t=1
Thread 1: Wait 500ms (replication lag buffer) t=2
Thread 1: Delete cache key again              t=3

Second delete catches any stale values written between
the first delete and the DB update completing.
```

---

## Real-World Example — ShopSphere Complete Cache Architecture

```java
// ShopSphere cache configuration
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaults = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();

        // Different TTLs per cache
        cacheConfigs.put("products",
            defaults.entryTtl(Duration.ofMinutes(30)));        // semi-static
        cacheConfigs.put("categories",
            defaults.entryTtl(Duration.ofHours(6)));           // rarely changes
        cacheConfigs.put("userProfiles",
            defaults.entryTtl(Duration.ofMinutes(15)));        // changes occasionally
        cacheConfigs.put("searchResults",
            defaults.entryTtl(Duration.ofMinutes(2)));         // stale quickly
        cacheConfigs.put("inventoryCount",
            defaults.entryTtl(Duration.ofSeconds(30)));        // changes rapidly
        cacheConfigs.put("orderStatus",
            defaults.entryTtl(Duration.ofMinutes(5)));         // changes over time

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaults)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}

// Product Service — full caching implementation
@Service
@Slf4j
public class ProductService {

    @Cacheable(
        value = "products",
        key = "#productId",
        unless = "#result == null"  // don't cache null results
    )
    public Product getProduct(String productId) {
        log.debug("Cache miss for product {}", productId);
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    @Cacheable(
        value = "products",
        key = "'list:' + #filter.category + ':' + #filter.page"
    )
    public ProductPage listProducts(ProductFilter filter) {
        return productRepository.findAll(filter);
    }

    @CachePut(value = "products", key = "#result.id")
    @CacheEvict(value = "products", key = "'list:*'", allEntries = true)
    public Product updateProduct(String productId, UpdateProductRequest request) {
        Product updated = productRepository.save(buildUpdated(productId, request));
        log.info("Product {} updated, cache refreshed", productId);
        return updated;
    }

    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        productRepository.deleteById(productId);
    }

    // Warm cache for top products on startup
    @PostConstruct
    public void warmCache() {
        log.info("Warming product cache...");
        productRepository.findTopProducts(100)
            .forEach(p -> redisTemplate.opsForValue()
                .set("products::" + p.getId(), p, Duration.ofMinutes(30)));
        log.info("Cache warm-up complete");
    }
}
```

**Cache metrics ShopSphere tracks:**
```java
@Scheduled(fixedDelay = 60_000)
public void reportCacheMetrics() {
    RedisInfo info = redisTemplate.execute(RedisServerCommands::info);

    double hitRatio = info.getKeyspaceHits() /
        (double)(info.getKeyspaceHits() + info.getKeyspaceMisses());

    meterRegistry.gauge("cache.hit.ratio",     hitRatio);
    meterRegistry.gauge("cache.memory.used",   info.getUsedMemory());
    meterRegistry.gauge("cache.keys.total",    info.getDbSize());
    meterRegistry.gauge("cache.evictions",     info.getEvictedKeys());

    if (hitRatio < 0.85) {
        alertService.warn("Cache hit ratio below 85%: " + hitRatio);
    }
}
```

---

## Interview Q&A

**Q: What is the difference between cache-aside and read-through caching?**
In cache-aside, the application explicitly checks the cache, handles misses by querying the database, and stores results in the cache. The application owns all cache logic. In read-through, the cache itself handles misses by fetching from the database automatically — the application only talks to the cache and never directly to the database. Cache-aside gives more control and is more resilient — a cache failure lets the application fall through to the DB directly. Read-through is cleaner code but tightly couples the cache to the database.

**Q: What is the difference between write-through and write-behind caching?**
Write-through updates both the cache and the database synchronously on every write — both are always consistent, write latency is higher but reads are always fast. Write-behind updates only the cache synchronously and flushes to the database asynchronously in the background — write latency is minimal but there is a data loss risk if the cache crashes before the async flush completes. Write-through is for consistency-critical data like sessions and user profiles. Write-behind is for high-frequency writes where brief data loss is acceptable, like shopping cart updates and view counters.

**Q: What is the Thundering Herd problem and how do you prevent it?**
When a popular cached item expires, all concurrent requests simultaneously see a cache miss and simultaneously query the database — potentially thousands of queries in milliseconds that overwhelm the DB. Prevention strategies are: a distributed lock so only one request queries the DB while others wait and retry, probabilistic early expiry where the cache is refreshed slightly before expiry with increasing probability as TTL decreases, and background refresh for the most popular items so they never actually expire from a user's perspective.

**Q: When would you choose LFU over LRU as a cache eviction policy?**
LRU evicts the least recently accessed item — it works well when access patterns have temporal locality, meaning recently used items will be used again soon. LFU evicts the least frequently accessed item — it works better when access patterns are heavily skewed, like a product catalogue where 10% of items get 90% of traffic. With LRU, a sudden burst of one-time accesses can evict genuinely popular items. With LFU, popular items accumulate high access counts and are protected from eviction. The practical consideration is that LFU requires frequency tracking overhead and can get stuck with stale popular items unless frequency counts decay over time.

**Q: How do you handle cache invalidation across multiple services in ShopSphere?**
When a product is updated, the Product Service must invalidate not just its own cache but also caches in the Search Service, the Recommendation Service, and any other service that caches product data. The pattern is to publish a ProductUpdated event to Kafka when a product changes. Each service subscribes to this event and evicts its local cache entries for that product. This avoids tight coupling — the Product Service does not need to know which services cache its data, and new services automatically receive invalidation events when they subscribe. The risk is a brief inconsistency window between the event publish and cache eviction in each subscriber.

---

Say **"next"** when ready for Topic 5 — Consistent Hashing.
