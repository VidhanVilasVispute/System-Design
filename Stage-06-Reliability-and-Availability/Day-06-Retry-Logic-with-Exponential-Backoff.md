# Stage 6 — Reliability & Availability
## Topic 6 : Retry Logic with Exponential Backoff

---

### Why Retry At All?

Not all failures are equal. Some are **transient** — they fix themselves in milliseconds.

```
Types of failures:

TRANSIENT (retry makes sense):
  ├── Network blip (packet loss, brief timeout)
  ├── DB connection pool momentarily exhausted
  ├── Service restarting (rolling deploy)
  ├── HTTP 429 Too Many Requests (rate limited)
  └── HTTP 503 Service Unavailable (brief overload)

PERMANENT (retry is useless / harmful):
  ├── HTTP 400 Bad Request (your data is wrong)
  ├── HTTP 401 Unauthorized (fix your token)
  ├── HTTP 404 Not Found (resource doesn't exist)
  ├── Validation failure (retrying won't fix bad input)
  └── Business logic error (insufficient balance)
```

> **Golden rule: Only retry on transient failures. Retrying permanent failures wastes resources and delays the inevitable error response.**

---

### Naive Retry — Why It's Dangerous

```java
// DANGEROUS — don't do this
for (int i = 0; i < 3; i++) {
    try {
        return paymentClient.charge(request);
    } catch (Exception e) {
        // immediately retry
    }
}
```

What happens when Payment Service is overloaded:

```
t=0s    1000 users hit Order Service simultaneously
t=0s    All 1000 call Payment Service → it struggles
t=0.1s  All 1000 fail → immediately retry (attempt 2)
t=0.1s  Payment Service hit with 1000 MORE requests
t=0.2s  All 1000 fail → immediately retry (attempt 3)
t=0.2s  Payment Service hit with 1000 MORE requests

Total calls to Payment = 3000 in 0.2 seconds
Payment Service was struggling with 1000 → now 3000 → DEAD 💀

This is called a RETRY STORM
```

---

### Exponential Backoff — The Fix

> **Wait longer between each retry attempt. Give the service time to recover.**

```
Attempt 1 fails → wait 1s  → retry
Attempt 2 fails → wait 2s  → retry
Attempt 3 fails → wait 4s  → retry
Attempt 4 fails → wait 8s  → retry
Attempt 5 fails → wait 16s → give up

Formula: wait = baseDelay × 2^(attemptNumber - 1)

base = 1s:
  Attempt 1 → 1 × 2⁰ = 1s
  Attempt 2 → 1 × 2¹ = 2s
  Attempt 3 → 1 × 2² = 4s
  Attempt 4 → 1 × 2³ = 8s
```

```
Timeline visual:

──●──────────●──────────────────●────────────────────────────────●──
  fail       fail               fail                             fail
  │←──1s────►│←──────2s────────►│←───────────4s────────────────►│
  attempt1   attempt2            attempt3                        give up
```

**Exponential backoff fixes retry storm — but not completely.**

---

### Jitter — The Missing Piece

Even with exponential backoff, if 1000 requests all fail at the same time, they all back off for exactly the same duration and **retry simultaneously again.**

```
Without jitter (thundering herd):

  t=0s    1000 requests fail
  t=1s    1000 requests retry simultaneously ← still a storm
  t=1s    all fail again
  t=3s    1000 requests retry simultaneously ← still a storm
```

**Jitter = add randomness to the wait time to spread retries out.**

```
With jitter:

  Request A: wait = 1s + random(0, 1s) = 1.3s
  Request B: wait = 1s + random(0, 1s) = 1.7s
  Request C: wait = 1s + random(0, 1s) = 1.1s
  ...

  t=0s    1000 requests fail
  t=1.1s  ~50 requests retry
  t=1.3s  ~100 requests retry
  t=1.5s  ~150 requests retry
  ...
  Spread across 1 second → service sees gentle ramp, not tsunami ✅
```

---

### Jitter Strategies — Which One to Use

#### Full Jitter (AWS Recommended)
```
wait = random(0, baseDelay × 2^attempt)

Attempt 1: random(0, 2s)   → e.g. 0.7s
Attempt 2: random(0, 4s)   → e.g. 2.1s
Attempt 3: random(0, 8s)   → e.g. 5.3s

Pros: Maximum spread, lowest load on recovering service
Cons: Can wait near 0s — might retry too quickly sometimes
Best for: High-concurrency systems, stampede prevention
```

#### Equal Jitter
```
halfCap = (baseDelay × 2^attempt) / 2
wait = halfCap + random(0, halfCap)

Attempt 1: 0.5 + random(0, 0.5) → between 0.5s and 1s
Attempt 2: 1.0 + random(0, 1.0) → between 1s   and 2s

Pros: Guarantees minimum wait time (avoids near-0 retries)
Cons: Less spread than full jitter
Best for: When minimum backoff matters
```

#### Decorrelated Jitter (Best Overall)
```
wait = min(maxDelay, random(baseDelay, previousWait × 3))

start: previousWait = baseDelay = 1s
  Attempt 1: random(1, 3)   = 2.1s  → previousWait = 2.1
  Attempt 2: random(1, 6.3) = 4.7s  → previousWait = 4.7
  Attempt 3: random(1, 14)  = 9.2s

Pros: Best distribution, no synchronization between clients
Best for: Production systems (AWS SDKs use this)
```

---

### Visual Comparison

```
No jitter:              ████████████████  (spike on every retry)
                        ████████████████
                        ████████████████

Full jitter:            ▓░░░░░░░░░░░░░░░  (spread evenly)
                           ▓░░░░░░░░░░░
                               ▓░░░░░░░

Decorrelated jitter:    ▓░░░░░░░░░░░░░░░  (grows organically)
                          ░▓░░░░░░░░░░░
                              ░░▓░░░░░░
```

---

### Max Retry Budget + Deadline Awareness

**Don't retry forever — set a budget:**

```
Max attempts: 3
Max total time: 5 seconds (deadline)

Attempt 1 at t=0s   → fail
Wait 1s
Attempt 2 at t=1s   → fail
Wait 2s
Attempt 3 at t=3s   → fail
Total time = 3s < 5s deadline → return error to caller

BUT if attempt 2 took 3 seconds:
  t=0s attempt 1 → fail (0.1s)
  t=1.1s attempt 2 → fail (3s)  ← slow response
  t=4.1s → only 0.9s left before deadline
  Should we retry? → NO. Not enough time budget left.
  → Return error immediately
```

**Deadline propagation concept (covered in Topic 8 deeply):**
```
Incoming request has deadline: 5 seconds total
  Order Service uses 0.5s
  Remaining: 4.5s → passed to Payment Service
  Payment Service uses 1s
  Remaining: 3.5s → available for retries
  If retries would exceed 3.5s → don't retry, fail fast
```

---

### Idempotency — Critical Prerequisite for Retrying

> **Before retrying, ask: is this operation safe to repeat?**

```
IDEMPOTENT operations (safe to retry):
  GET  /products/123     → same result every time ✅
  PUT  /orders/123/cancel → cancelling twice = still cancelled ✅

NON-IDEMPOTENT (dangerous to retry):
  POST /payments/charge  → charging twice = double charge 💀
  POST /orders           → creating twice = duplicate order 💀
```

**How to make POST idempotent — Idempotency Key:**

```
Client generates a unique key per request:
  idempotency-key: "order-req-uuid-abc123"

First call:
  POST /payments/charge
  idempotency-key: abc123
  → Server processes, stores result with key abc123
  → Returns: { status: SUCCESS, txnId: TXN-999 }

Retry call (network blip):
  POST /payments/charge
  idempotency-key: abc123   ← same key
  → Server sees key abc123 already processed
  → Returns same cached result: { status: SUCCESS, txnId: TXN-999 }
  → NO double charge ✅
```

```
ShopSphere Payment Service idempotency table:

  ┌──────────────┬──────────────┬────────────┬──────────┐
  │ idempotency_ │   request_   │  response_ │ expires_ │
  │ key          │   hash       │  body      │ at       │
  ├──────────────┼──────────────┼────────────┼──────────┤
  │ abc123       │ sha256(body) │ {SUCCESS}  │ +24hrs   │
  └──────────────┴──────────────┴────────────┴──────────┘
```

---

### Resilience4j Retry — Production Config

```yaml
# application.yml — ShopSphere
resilience4j:
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 1s                    # base wait
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2     # 1s → 2s → 4s
        exponentialMaxWaitDuration: 10s     # cap at 10s
        enableRandomizedWait: true          # add jitter
        randomizedWaitFactor: 0.5           # ±50% randomness

        # Only retry on these
        retryExceptions:
          - java.io.IOException
          - feign.RetryableException
          - org.springframework.web.client.ResourceAccessException

        # Never retry on these
        ignoreExceptions:
          - com.shopsphere.exception.ValidationException
          - com.shopsphere.exception.InsufficientFundsException
          - feign.FeignException.BadRequest       # 400
          - feign.FeignException.Unauthorized     # 401
```

**Service code:**
```java
@Service
public class OrderService {

    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        // Attach idempotency key — safe to retry
        return paymentClient.charge(request,
            request.getIdempotencyKey());  // header forwarded via Feign
    }

    private PaymentResponse paymentFallback(PaymentRequest request,
                                             Exception ex) {
        // All retries exhausted AND circuit open → queue for async
        eventPublisher.publishEvent(new PaymentRetryExhaustedEvent(request));
        return PaymentResponse.pending(request.getOrderId());
    }
}
```

**Retry event listener for observability:**
```java
@EventListener
public void onRetry(RetryOnRetryEvent event) {
    log.warn("Retry attempt {} for [{}]. Last error: {}",
        event.getNumberOfRetryAttempts(),
        event.getName(),
        event.getLastThrowable().getMessage());

    // Expose to Micrometer / Prometheus
    meterRegistry.counter("retry.attempts",
        "service", event.getName()).increment();
}
```

---

### The Complete Retry Decision Tree

```
Request fails
      │
      ▼
Is it a permanent error? (4xx except 429)
      │ YES → return error immediately, don't retry
      │ NO
      ▼
Is circuit breaker OPEN?
      │ YES → return fallback immediately
      │ NO
      ▼
Have we exceeded max attempts?
      │ YES → return fallback / error
      │ NO
      ▼
Is there enough deadline budget left?
      │ NO → return error immediately
      │ YES
      ▼
Is the operation idempotent (or has idempotency key)?
      │ NO → log warning, don't retry POST without key
      │ YES
      ▼
Calculate wait = exponentialBackoff + jitter
Wait...
Retry attempt
```

---

### ShopSphere Retry Strategy Map 🛒

```
┌────────────────────────────────────────────────────────┐
│             ShopSphere Retry Configuration              │
│                                                        │
│  Order → Payment       maxAttempts=3, backoff=1s×2     │
│                        jitter=full, idempotencyKey=✅   │
│                        ignores: 400, 401, 402          │
│                                                        │
│  Order → Inventory     maxAttempts=2, backoff=500ms×2  │
│                        GET only → always idempotent    │
│                                                        │
│  Order → Notification  maxAttempts=1 (fire and forget) │
│                        Kafka handles retry separately  │
│                                                        │
│  Search → Elasticsearch maxAttempts=2, backoff=200ms   │
│                        fallback: DB query              │
│                                                        │
│  Any → Redis           maxAttempts=2, backoff=100ms    │
│                        fallback: skip cache, hit DB    │
└────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. Why is exponential backoff alone not enough — what does jitter add?**
> Without jitter, all clients that failed simultaneously back off for the same duration and retry simultaneously again — creating a thundering herd on every retry wave. Jitter spreads retries randomly across time, converting synchronized spikes into a gentle ramp that the recovering service can handle.

**Q2. What's a retry storm and how do you prevent it?**
> A retry storm is when a large number of clients simultaneously retry a failing service, amplifying load on an already-struggling dependency. Prevention: exponential backoff to reduce retry frequency, jitter to desynchronize retries, circuit breaker to stop retrying entirely once failure rate is too high, and max retry budget to cap total retry attempts.

**Q3. Why must you check idempotency before retrying a POST?**
> POST requests typically create resources. Retrying a non-idempotent POST can create duplicates — double charges, duplicate orders. Before retrying, either confirm the operation is idempotent by design (PUT/DELETE), or attach an idempotency key so the server can detect and deduplicate replayed requests.

**Q4. Should retry logic sit inside or outside the circuit breaker?**
> Retry sits inside the circuit breaker. If retry wraps the circuit breaker, retries happen after the breaker has already decided to block, wasting time. If the circuit breaker wraps retry, each retry attempt is a call the breaker evaluates — if retries keep failing, the breaker accumulates failures and trips as intended. Correct order: CircuitBreaker → Retry → TimeLimiter → actual call.

**Q5. How do you decide the max retry count and backoff values?**
> Work backwards from your SLO and timeout budget. If the caller's total timeout is 5 seconds, and each call can take up to 1 second, you have budget for 2–3 retries with backoff. Start conservative: 3 attempts, 1s base, 2× multiplier, 10s cap. Monitor retry rate in production — if retries are rare, the config is right. If retries are frequent, the upstream has a reliability problem worth fixing directly.

---

### One-Line Summary

> **Retry handles transient failures — but only with exponential backoff (grow the wait) + jitter (randomize it) + idempotency keys (make it safe) + circuit breaker (know when to stop) — otherwise your retries become the attack.**

---

Ready for **Topic 7: Bulkhead Pattern** — isolate thread pools per dependency so one slow service can't starve everything else, and how it maps directly to ShopSphere's service boundaries?

Type **next** 🚀
