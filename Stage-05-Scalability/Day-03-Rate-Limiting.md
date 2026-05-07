# Topic 3 — Rate Limiting

## Why Rate Limiting Exists

Without rate limiting, a single bad actor (or a bug) can take down your entire system:

```
ShopSphere without rate limiting:

Scenario 1 — Malicious:
  Attacker writes a script → 50,000 POST /orders per second
  order-service → overwhelmed → DB connection pool exhausted
  → every real user gets 503

Scenario 2 — Buggy client:
  Mobile app has infinite retry loop bug
  10,000 users hit the bug simultaneously
  → same result, unintentional DDoS

Scenario 3 — Scraper:
  Competitor scrapes GET /products every 10ms
  → product-service CPU pegged → legitimate users slow

Rate limiting = protect your system from being overwhelmed,
                regardless of who's causing it.
```

---

## The Three Algorithms

### Algorithm 1 — Fixed Window

Divide time into fixed buckets. Count requests per bucket. Reject when limit exceeded.

```
Limit: 5 requests per minute

Timeline:
  |-------- Window 1 --------|-------- Window 2 --------|
  0s                         60s                        120s

  Requests arrive:
  t=10s → req 1 ✅  count=1
  t=20s → req 2 ✅  count=2
  t=30s → req 3 ✅  count=3
  t=40s → req 4 ✅  count=4
  t=50s → req 5 ✅  count=5
  t=55s → req 6 ❌  count=5 (limit hit)
  t=60s → WINDOW RESETS → count=0
  t=61s → req 7 ✅  count=1
```

**Implementation (conceptual):**
```
key   = "rate:userId:42:window:1710000"  ← epoch minute
count = INCR key
if count == 1: EXPIRE key 60
if count > limit: reject
```

### The Fatal Flaw — Boundary Burst Attack

```
Limit: 5 requests per minute

  Window 1                    Window 2
  |__________________________|__________________________|
                    55s      60s      65s

  t=55s → 5 requests ✅  (end of window 1, count=5)
  t=60s → window resets
  t=65s → 5 requests ✅  (start of window 2, count=5)

  Result: 10 requests in 10 seconds — DOUBLE the intended limit!
  
  An attacker who knows your window boundaries can burst 2x every window.
```

Fixed window is simple and cheap but has this boundary vulnerability.

---

### Algorithm 2 — Sliding Window

Instead of fixed buckets, look at a **rolling window** of the last N seconds.

```
Limit: 5 requests per 60 seconds

At t=65s, the window looks back to t=5s:

  Timeline:
  ──────────────────────────────────────────────────────────▶
  t=5   t=10  t=20  t=30  t=40  t=50  t=55  t=60  t=65
        req1  req2  req3  req4  req5        req6  CHECK

  At t=65s, sliding window [5s → 65s] sees:
  req1(t=10), req2(t=20), req3(t=30), req4(t=40), req5(t=50)
  Count = 5 → req6 at t=65 → REJECTED ❌

  At t=71s, sliding window [11s → 71s]:
  req1(t=10) falls OUT of window
  Count = 4 → next request ALLOWED ✅
```

**Implementation using Redis Sorted Set:**
```
key    = "rate:userId:42"
now    = current timestamp (milliseconds)
window = 60,000ms

ZREMRANGEBYSCORE key 0 (now - window)   ← remove old entries
count = ZCARD key                        ← count remaining
if count >= limit: reject
ZADD key now now                         ← add current request
EXPIRE key 60                            ← cleanup TTL
```

```
Why Sorted Set?
  Score  = timestamp
  Member = timestamp (unique per request)

  ZREMRANGEBYSCORE removes everything older than window.
  ZCARD gives exact count of requests in the last N seconds.
  Precise, no boundary exploit.
```

**The cost:** More memory than fixed window (stores every request timestamp). For high-traffic systems, this gets expensive.

---

### Algorithm 3 — Token Bucket ⭐ (Most Important)

This is the most widely used algorithm in production. Understand it deeply.

```
Mental Model:
  A bucket has a maximum capacity of N tokens.
  Tokens are added at a fixed rate (e.g., 10 tokens/second).
  Each request consumes 1 token.
  If bucket is empty → request rejected.
  Bucket never exceeds capacity.

  ┌─────────────────────────────┐
  │         Token Bucket        │
  │  Capacity: 10 tokens        │
  │  Refill: 2 tokens/second    │
  │                             │
  │  [●][●][●][●][●][ ][ ][ ]  │  ← 5 tokens currently
  │                             │
  │  Request arrives → takes 1  │
  │  Empty? → REJECT            │
  └─────────────────────────────┘
```

**Simulation:**
```
Bucket capacity = 10, refill rate = 2/sec

t=0s:   bucket = 10 (full)
t=0s:   5 requests burst → bucket = 5  ✅ all allowed
t=1s:   +2 tokens → bucket = 7
t=1s:   3 requests → bucket = 4  ✅
t=2s:   +2 tokens → bucket = 6
t=2s:   7 requests → 6 allowed ✅, 1 rejected ❌

Key insight: Burst IS allowed (up to bucket capacity).
             Sustained rate is capped (by refill rate).
```

**This is the key distinction:**
```
Fixed Window:   Hard limit per window. No burst allowed across boundary.
Sliding Window: Smooth limit. No burst at all.
Token Bucket:   Burst allowed up to capacity. Sustained rate capped.
                → Models real user behavior realistically.
```

**Implementation (lazy evaluation — no background thread needed):**
```
Instead of actually adding tokens every second,
calculate how many tokens SHOULD have accumulated since last request.

State stored in Redis:
  tokens    = current token count
  last_refill = timestamp of last request

On each request:
  now           = current time
  elapsed       = now - last_refill
  new_tokens    = elapsed * refill_rate
  tokens        = min(capacity, tokens + new_tokens)
  last_refill   = now

  if tokens >= 1:
      tokens -= 1
      ALLOW ✅
  else:
      REJECT ❌

Store {tokens, last_refill} atomically in Redis.
```

---

## Side-by-Side Comparison

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────────┐
│                  │  Fixed Window    │  Sliding Window  │  Token Bucket    │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Burst handling   │ Allows 2x burst  │ No burst         │ Controlled burst │
│                  │ at boundaries ❌  │                  │ up to capacity ✅ │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Memory           │ O(1) per user    │ O(requests)      │ O(1) per user    │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Accuracy         │ Approximate      │ Exact            │ Approximate      │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Implementation   │ Trivial          │ Moderate         │ Moderate         │
├──────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Used by          │ Simple APIs      │ High-precision   │ AWS, Stripe,     │
│                  │                  │ systems          │ Nginx, Cloudflare│
└──────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

---

## Distributed Rate Limiting — The Hard Problem

This is where it gets interesting for microservices.

### The Problem

```
Single instance — trivial:
  1 rate limiter, 1 counter in memory. Done.

3 instances behind a load balancer:
  Limit: 10 req/min per user

  svc-1 counter: userId=42 → 4 requests
  svc-2 counter: userId=42 → 4 requests
  svc-3 counter: userId=42 → 4 requests

  Total real requests: 12 → OVER LIMIT
  But each instance sees only 4 → all allowed ❌

  Each instance has no idea what the others are doing.
```

### Solution: Centralized Redis Counter

```
All instances talk to the same Redis:

  svc-1 ──▶ ┌────────────────────────────┐
  svc-2 ──▶ │   Redis                    │
  svc-3 ──▶ │   rate:user:42 = 9         │
             │   (atomic INCR)            │
             └────────────────────────────┘

  INCR is atomic in Redis — no race condition.
  All instances share the same counter.
  When counter hits 10 → all instances reject for this user.
```

**Lua script for atomic token bucket in Redis:**
```lua
-- Atomic: check + update in one Redis round trip
local key       = KEYS[1]
local capacity  = tonumber(ARGV[1])
local refill    = tonumber(ARGV[2])   -- tokens per second
local now       = tonumber(ARGV[3])   -- current epoch ms

local bucket    = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens    = tonumber(bucket[1]) or capacity
local last      = tonumber(bucket[2]) or now

local elapsed   = (now - last) / 1000.0
local refilled  = math.min(capacity, tokens + elapsed * refill)

if refilled >= 1 then
    redis.call('HMSET', key, 'tokens', refilled - 1, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return 1   -- ALLOW
else
    redis.call('HMSET', key, 'tokens', refilled, 'last_refill', now)
    return 0   -- REJECT
end
```

Why Lua? Redis executes it **atomically** — no other command runs between the read and write. No race condition across multiple app instances.

---

## Where Rate Limiting Lives in ShopSphere

```
Client
  │
  ▼
┌─────────────────────────────────────────┐
│           API Gateway                   │  ← BEST place for rate limiting
│                                         │    One choke point, all traffic
│  Per-user:   100 req/min  (JWT userId)  │    passes through here
│  Per-IP:     300 req/min  (unauthenticated)
│  Per-route:  POST /orders → 10 req/min │    (order spam prevention)
│                                         │
│  Redis ←───────────────────────────────┤
└─────────────────────────────────────────┘
  │
  ▼
Downstream services (order-svc, product-svc, etc.)
  ← already protected, don't need their own rate limiting
     (but can add service-level limits for defense in depth)
```

**Spring Cloud Gateway — Rate Limiting Config:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10    # tokens/sec
                redis-rate-limiter.burstCapacity: 20    # bucket size
                redis-rate-limiter.requestedTokens: 1
                key-resolver: "#{@userKeyResolver}"     # rate limit per user
```

```java
@Bean
KeyResolver userKeyResolver() {
    return exchange -> Mono.justOrEmpty(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    ).defaultIfEmpty("anonymous");
}
```

This uses **token bucket** via Spring's built-in Redis integration — exactly what we just studied.

---

## Rate Limit Response — What to Return

```
HTTP 429 Too Many Requests

Headers:
  X-RateLimit-Limit:     100        ← max allowed
  X-RateLimit-Remaining: 0          ← tokens left
  X-RateLimit-Reset:     1710000060 ← epoch when bucket refills
  Retry-After:           30         ← seconds until retry

Body:
  {
    "error": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 30 seconds.",
    "retryAfter": 30
  }
```

Good clients (SDKs, mobile apps) read `Retry-After` and back off automatically.

---

## Levels of Rate Limiting

```
Level             Key               Use Case
──────────────────────────────────────────────────────────
IP-based          IP address        Block anonymous DDoS, scrapers
User-based        JWT userId        Fair use per user
API-key based     API key           Tiered pricing (free/pro/enterprise)
Endpoint-based    route + userId    POST /orders stricter than GET /products
Service-to-svc    serviceId         Prevent one microservice flooding another
Global            system-wide       Emergency circuit breaker
```

ShopSphere can layer these — global IP limit at the edge (Nginx/CDN), user-level token bucket at the gateway, endpoint-level limits on sensitive routes.

---

## Interview Angles

**Q: Explain token bucket vs sliding window.**
> Token bucket allows controlled bursts up to the bucket capacity, then limits sustained rate by the refill rate. It models real user behavior — a user can fire 10 requests quickly then slows down. Sliding window is more precise — exactly N requests in the last N seconds, no bursts. Token bucket is more common in production because O(1) memory and burst tolerance.

**Q: How do you implement rate limiting across multiple service instances?**
> Centralize the counter in Redis. Redis's INCR is atomic, so all instances share the same state. For token bucket, use a Lua script to read-compute-write atomically in a single round trip, eliminating race conditions between instances.

**Q: Why use Lua scripts in Redis for rate limiting?**
> Multiple Redis commands (GET, compute, SET) are not atomic individually — a race condition between two instances could allow more requests than the limit. A Lua script executes as a single atomic unit in Redis. No other command runs between the read and the write.

**Q: Where should rate limiting live in a microservices architecture?**
> At the API Gateway — single entry point, all traffic passes through, one place to configure and enforce limits. Downstream services don't need their own rate limiting (though defense-in-depth is fine). Centralizing at the gateway also means one Redis backing store for all distributed counters.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  Fixed Window    │ Simple, cheap, boundary burst vulnerability        │
│  Sliding Window  │ Precise, expensive memory, no burst allowed        │
│  Token Bucket    │ Burst-tolerant, O(1) memory, industry standard ✅  │
├──────────────────────────────────────────────────────────────────────┤
│  Distributed     │ Central Redis counter + atomic Lua script          │
│  In ShopSphere   │ Spring Cloud Gateway + Redis token bucket          │
│                  │ Per-user (X-User-Id) + per-IP + per-route          │
└──────────────────────────────────────────────────────────────────────┘
```

---

Rate limiting done — solid foundation. Next is **Topic 4 — Single Point of Failure (SPOF)**: how to identify every SPOF in a system and the redundancy patterns that eliminate them. Ready?
