# Topic 6 — Thundering Herd Problem

## What Is It?

```
The Thundering Herd:
  A large number of processes or threads wake up simultaneously
  to compete for a resource — most of them will fail or be wasted,
  and the stampede itself causes the damage.

In distributed systems, the most common form:
  Cache expires → thousands of requests hit the DB simultaneously
  → DB gets overwhelmed → slow queries → more timeouts
  → more retries → DB dies → cascading failure
```

---

## The Setup — Why Caching Creates This Problem

```
Normal operation (cache warm):

  10,000 users browsing ShopSphere homepage
        │
        ▼
  GET /products/featured
        │
        ▼
  ┌─────────────────┐
  │   Redis Cache   │  HIT ✅ → returns in 1ms
  │   key: featured │  10,000 requests served from cache
  │   TTL: 300s     │  DB sees: 0 requests
  └─────────────────┘
```

```
Cache expires (TTL = 300s hits zero):

  t=300s: Redis deletes "featured" key

  ALL 10,000 in-flight and incoming requests:
        │
        ▼
  Redis: MISS ❌ → all 10,000 fall through to DB simultaneously

  ┌──────────────────────────────────────────────────────┐
  │              PostgreSQL                              │
  │                                                      │
  │  Connection pool: 100 connections max                │
  │  Suddenly: 10,000 threads trying to connect          │
  │                                                      │
  │  SELECT * FROM products WHERE featured = true        │
  │  × 10,000 concurrent identical queries               │
  │                                                      │
  │  CPU: 100% 🔥                                        │
  │  Connection queue: overflowing                       │
  │  Query time: 1ms → 30s (contention)                  │
  │  Connection timeouts → exceptions in app             │
  │  App retries → MORE requests to DB                   │
  │  DB dies                                             │
  └──────────────────────────────────────────────────────┘
```

The cruel irony: **the cache was working perfectly, protecting the DB. Its expiry is what causes the disaster.**

---

## Why It's Worse Than You Think — The Retry Amplification

```
t=300s:  Cache expires
t=300s:  10,000 requests hit DB
t=302s:  DB slow → requests timeout after 2s
t=302s:  Client retries (Feign client has retry logic) → 20,000 requests
t=304s:  20,000 requests timeout → retry again → 40,000 requests
t=306s:  DB completely unresponsive

Each retry doubles the load. Exponential growth.
Without retry limits + jitter, you're making it worse trying to fix it.
```

---

## Solution 1 — Cache Locking (Mutex / Single Population)

Only **one thread** is allowed to repopulate the cache. All others wait.

```
Thread 1: cache miss → acquires lock → queries DB → populates cache → releases lock
Thread 2: cache miss → tries lock → BLOCKED → waits
Thread 3: cache miss → tries lock → BLOCKED → waits
...
Thread N: cache miss → tries lock → BLOCKED → waits

Lock released → Thread 2 checks cache → HIT ✅ (Thread 1 populated it)
             → Thread 3 checks cache → HIT ✅
             → All N threads get result from cache, no DB hit

DB sees: exactly 1 query instead of N queries ✅
```

**Implementation with Redis distributed lock:**

```java
public List<Product> getFeaturedProducts() {
    String cacheKey  = "products:featured";
    String lockKey   = "lock:products:featured";
    String lockValue = UUID.randomUUID().toString();  // unique per thread

    // Try to get from cache first
    List<Product> cached = redis.get(cacheKey);
    if (cached != null) return cached;

    // Cache miss — try to acquire lock
    Boolean acquired = redis.set(
        lockKey, lockValue,
        SetArgs.Builder.nx().ex(10)  // NX = only if not exists, EX = 10s TTL
    );

    if (acquired) {
        try {
            // Double-check: another thread may have populated while we waited
            cached = redis.get(cacheKey);
            if (cached != null) return cached;

            // We have the lock — query DB
            List<Product> products = productRepository.findFeatured();

            // Populate cache
            redis.setex(cacheKey, 300, products);
            return products;

        } finally {
            // Release lock — only if WE own it (check lockValue)
            releaseLock(lockKey, lockValue);
        }
    } else {
        // Another thread has the lock — wait and retry
        Thread.sleep(50);  // back off
        return getFeaturedProducts();  // retry — will hit cache now
    }
}
```

```
Why check lockValue on release?
  Thread 1 acquires lock (lockValue = "abc")
  Thread 1 is slow — lock TTL expires (10s)
  Thread 2 acquires lock (lockValue = "xyz")
  Thread 1 finishes — tries to release lock
  Without value check: Thread 1 deletes Thread 2's lock → two threads in critical section
  With value check: Thread 1 sees lockValue ≠ "abc" → skips release → safe ✅
```

**The downside:** threads are blocked waiting. Under extreme load, many threads pile up waiting → memory pressure. Acceptable for most cases, but not pure zero-wait.

---

## Solution 2 — Probabilistic Early Expiry (XFetch)

Instead of waiting for TTL to hit zero, **proactively refresh** the cache before it expires — with probability increasing as expiry approaches.

```
The insight:
  If TTL = 300s and we're at t=295s (5s left),
  there's a HIGH probability of thundering herd incoming.
  One request should refresh NOW, while cache is still valid
  and other requests still get cache hits.

Formula (XFetch algorithm):
  current_time - (delta × beta × log(rand())) > expiry_time
  
  delta = time it took to compute the value (DB query duration)
  beta  = tuning constant (typically 1.0)
  rand  = random number between 0 and 1

  As expiry_time approaches:
    left side grows more likely to exceed right side
    → more requests decide to refresh early
    One gets there first, refreshes, resets TTL
    Others still get cache hits
```

```
Practical simplified version:

  TTL_THRESHOLD = 30s  (refresh if less than 30s remaining)

  ttl = redis.ttl(cacheKey)

  if (ttl < 0) {
      // Already expired → full cache miss, use locking
  } else if (ttl < TTL_THRESHOLD && random() < 0.1) {
      // 10% of requests trigger early refresh when < 30s left
      // While this request refreshes in background,
      // ALL other requests still get the old cached value
      asyncRefresh(cacheKey);
      return cachedValue;  // return current (slightly stale) value
  }
  return cachedValue;  // normal cache hit
```

```
Timeline:
  t=270s: TTL = 30s remaining. 10% of requests trigger async refresh.
  t=271s: One request queries DB (async), repopulates cache, new TTL = 300s.
  t=300s: Old TTL = 0. But cache was already refreshed at t=271s! 
          No miss ever occurs.

DB sees: 1 background query at t=271s
         0 queries at t=300s (the dangerous moment)
```

---

## Solution 3 — Staggered TTLs (Jitter)

The thundering herd also occurs when you populate the cache for many keys simultaneously — they all expire at the same time.

```
BAD — synchronized TTLs:

  On startup (or after cache flush):
    Cache product:1  TTL = 300s  ← expires at t=300s
    Cache product:2  TTL = 300s  ← expires at t=300s
    Cache product:3  TTL = 300s  ← expires at t=300s
    ... (10,000 product keys)

  t=300s: ALL 10,000 keys expire simultaneously → 10,000 DB queries

GOOD — jittered TTLs:

  TTL = BASE_TTL + random(0, JITTER)
  
  Cache product:1  TTL = 300 + 47 = 347s
  Cache product:2  TTL = 300 + 12 = 312s
  Cache product:3  TTL = 300 + 89 = 389s

  Expirations spread over a 100s window
  DB sees: ~100 queries/second instead of 10,000 at once ✅
```

```java
// In ShopSphere product-service:
private static final int BASE_TTL   = 300;   // seconds
private static final int TTL_JITTER = 60;    // ± range

public void cacheProduct(Product product) {
    int ttl = BASE_TTL + ThreadLocalRandom.current().nextInt(TTL_JITTER);
    redis.setex("product:" + product.getId(), ttl, product);
}
```

---

## Solution 4 — Background Refresh (Cache-Aside with Async Reload)

Never let the cache actually expire. A background job refreshes it before it can.

```
┌────────────────────────────────────────────────────────┐
│  Cache populated at t=0 with TTL=300s                  │
│                                                        │
│  Scheduled job runs every 250s:                        │
│    → Queries DB for featured products                  │
│    → Writes to cache with fresh TTL=300s               │
│    → Cache NEVER expires from TTL perspective          │
│                                                        │
│  Requests always hit cache (cache always warm)         │
│  DB sees: 1 query every 250s from the refresh job      │
└────────────────────────────────────────────────────────┘

Timeline:
  t=0:    Cache populated. TTL=300s.
  t=250s: Background job refreshes. TTL reset to 300s.
  t=300s: Would have been expiry — but TTL is already 300s again. ✅
  t=500s: Background job refreshes again. TTL reset.
  ... repeats forever

The cache never gets a chance to expire.
```

```java
// In ShopSphere product-service:
@Scheduled(fixedRate = 250_000)  // every 250s
public void refreshFeaturedProductsCache() {
    try {
        List<Product> products = productRepository.findFeatured();
        redis.setex("products:featured", 300, products);
        log.info("Featured products cache refreshed");
    } catch (Exception e) {
        log.error("Cache refresh failed — old cache still serving", e);
        // Don't crash — stale cache is better than no cache
    }
}
```

**Trade-off:** Works great for well-known, predictable keys (featured products, homepage data). Doesn't work for user-specific or unpredictable keys (you can't pre-warm 10M user caches).

---

## Solution 5 — Request Coalescing

At the infrastructure level, multiple identical in-flight requests are collapsed into one.

```
10,000 simultaneous:  GET /products/featured

Without coalescing:
  10,000 cache misses → 10,000 DB queries

With coalescing (e.g., Nginx proxy_cache_lock):
  Request 1 → cache miss → sent to upstream (DB/service)
  Requests 2-9999 → cache miss → HELD at proxy, waiting for Request 1
  Request 1 returns → proxy caches result
  Requests 2-9999 → served from cache instantly

  DB sees: 1 query
```

```nginx
proxy_cache_lock on;           # hold duplicate requests
proxy_cache_lock_timeout 5s;   # max wait time
proxy_cache_use_stale updating; # serve stale while refreshing
```

Nginx does this natively. Varnish, Cloudflare, and most CDNs do it too.

---

## The Thundering Herd at Service Level (Not Just Cache)

Cache expiry is the most common form, but the pattern appears elsewhere:

### Retry storms:
```
order-service calls payment-service
payment-service is slow (GC pause)
order-service timeout → retry immediately
100 threads all retry simultaneously
payment-service now has 200 requests instead of 100
→ GC pause gets worse → more timeouts → more retries
→ death spiral

Fix: Exponential backoff + jitter on retries
  retry delay = min(BASE * 2^attempt + random(0, 100ms), MAX)
  
  attempt 1: wait 100ms + jitter
  attempt 2: wait 200ms + jitter
  attempt 3: wait 400ms + jitter
  ...
  Jitter prevents all retrying threads from retrying simultaneously.
```

### Cold start / deployment:
```
New instance of product-service starts up.
Cache is empty (new pod, fresh Redis connection).
First N requests all miss cache → hit DB.
→ N DB queries simultaneously on startup.

Fix: Warm the cache on startup before accepting traffic.
  @EventListener(ApplicationReadyEvent.class)
  public void warmCache() {
      List<Product> featured = productRepository.findFeatured();
      redis.setex("products:featured", 300, featured);
      // now mark pod as ready (readiness probe passes)
  }
  
  K8s readiness probe fails until warmup completes
  → LB doesn't route traffic to pod until cache is warm
```

---

## ShopSphere Mapping

```
Thundering herd risks in ShopSphere:

1. products:featured key expires
   → 10,000 homepage requests hit product-service → DB overwhelmed
   Fix: Background refresh every 250s + jittered TTL

2. Flash sale starts (scheduled event):
   → All product cache keys for sale items expire simultaneously
   → Coordinated thundering herd
   Fix: Staggered TTLs at population time + cache lock on miss

3. Deployment / pod restart:
   → New order-service pod starts, no local state (we're stateless ✅)
   → Redis still has all cache → no thundering herd here
   → But if Redis itself restarts (cache flush) → full miss storm
   Fix: Redis persistence (AOF) + Redis Sentinel (no cold restart)

4. Kafka consumer group rebalance:
   → All consumers stop, rebalance, restart simultaneously
   → All consumers start replaying from same offset → burst of DB writes
   Fix: Staggered consumer startup + idempotency (dedup by eventId)
```

---

## Solution Selector

```
Scenario                              Best Solution
────────────────────────────────────────────────────────────────
Single hot key expires                Cache locking (mutex)
Many keys expire at same time         Jittered TTLs
Predictable, global data              Background refresh (never expire)
CDN / proxy level                     Request coalescing (Nginx)
Retry storms                          Exponential backoff + jitter
Cold start                            Cache warming on readiness
Unknown key set (user-specific)       Probabilistic early expiry (XFetch)
```

---

## Interview Angles

**Q: What is the thundering herd problem?**
> When a cache entry expires, all concurrent requests experience a miss simultaneously and fall through to the database. The DB receives N identical queries at once — for high-traffic systems, this can be thousands of queries — overwhelming it and causing cascading failures.

**Q: How do you solve it?**
> Several complementary strategies: cache locking ensures only one thread repopulates while others wait; jittered TTLs prevent synchronized expiry across many keys; background refresh proactively updates the cache before it expires; probabilistic early expiry refreshes it while still valid. In production, you typically combine jitter (always) with locking (for critical keys) and background refresh (for global/predictable keys).

**Q: Why add jitter to retry delays?**
> Without jitter, all failed requests retry at exactly the same time (BASE × 2^attempt). You replace one thundering herd with another at each retry interval. Jitter adds randomness, spreading retries over a time window so the downstream service gets a manageable trickle rather than synchronized bursts.

**Q: What's the double-check pattern in cache locking and why is it needed?**
> After acquiring the lock, you check the cache again before querying the DB. While you were waiting to acquire the lock, the thread that held it may have already populated the cache. Without the double-check, you'd query the DB redundantly even though the cache is now warm.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  What      │ Cache expires → N simultaneous misses → DB overwhelmed  │
├──────────────────────────────────────────────────────────────────────┤
│  Why bad   │ Retry amplification. Each retry doubles the load.       │
│            │ DB connection pool exhausted. Cascading failure.        │
├──────────────────────────────────────────────────────────────────────┤
│  Solutions │ 1. Cache lock      — 1 thread repopulates, rest wait    │
│            │ 2. XFetch          — probabilistic early refresh        │
│            │ 3. TTL jitter      — stagger expirations at write time  │
│            │ 4. Background job  — refresh before expiry, proactive   │
│            │ 5. Coalescing      — proxy collapses duplicate requests  │
├──────────────────────────────────────────────────────────────────────┤
│  Also      │ Retry storms (fix: exp backoff + jitter)                │
│  appears   │ Cold start (fix: cache warm before readiness probe)     │
├──────────────────────────────────────────────────────────────────────┤
│  ShopSphere│ Jitter all TTLs. Background refresh for hot global keys.│
│            │ Cache warming on pod startup. Redis persistence.        │
└──────────────────────────────────────────────────────────────────────┘
```

---

Six topics deep. Next is **Topic 7 — Back-pressure**: how a drowning downstream service signals upstream to slow down — and why without it, fast producers will always kill slow consumers. Ready?
