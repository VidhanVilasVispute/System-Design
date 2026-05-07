# Stage 6 — Reliability & Availability
## Topic 7 : Bulkhead Pattern

---

### The Name — Where It Comes From

```
A ship's hull is divided into watertight compartments
called BULKHEADS.

  ┌────┬────┬────┬────┬────┐
  │    │    │////│    │    │
  │ OK │ OK │HOLE│ OK │ OK │
  │    │    │////│    │    │
  └────┴────┴────┴────┴────┘

One compartment floods → ship stays afloat.
Without bulkheads → entire ship sinks.

Same principle in software:
One dependency drowns → only its pool is affected.
Without bulkheads → entire service drowns.
```

---

### The Problem — Thread Pool Exhaustion

Every incoming request to your service occupies a thread.
If a downstream dependency is slow — that thread **waits**.

```
Order Service — single shared thread pool (200 threads)

Dependency A: Payment    → responds in 50ms  ✅
Dependency B: Inventory  → responds in 50ms  ✅
Dependency C: Notification → suddenly slow   ⚠️
              (responding in 30 seconds)

Traffic: 100 req/s, each touches Notification

t=0s    100 threads waiting on Notification
t=1s    200 threads waiting on Notification
t=2s    Thread pool FULL ← all 200 threads stuck

t=2s    New request comes in → needs Payment (fast!)
        → No threads available
        → Request REJECTED or queued
        → Payment calls start failing too 💀

t=2s    Order Service is effectively DOWN
        because of ONE slow dependency
```

**Notification being slow killed Payment, Inventory, everything.**

---

### The Bulkhead Solution — Isolated Thread Pools

```
Order Service — SEPARATE thread pool per dependency

  ┌─────────────────────────────────────────────┐
  │              Order Service                  │
  │                                             │
  │  ┌──────────────┐   ┌──────────────┐        │
  │  │Payment Pool  │   │Inventory Pool│        │
  │  │ 50 threads   │   │ 30 threads   │        │
  │  └──────┬───────┘   └──────┬───────┘        │
  │         │                  │                │
  │  ┌──────────────┐   ┌──────────────┐        │
  │  │Notification  │   │Search Pool   │        │
  │  │Pool 20 threads│   │ 20 threads   │        │
  │  └──────┬───────┘   └──────┬───────┘        │
  └─────────┼──────────────────┼────────────────┘
            │                  │
      Notification          Search
      (slow/dead)           (healthy)

Notification pool fills up → only 20 threads affected
Payment pool unaffected   → still serving 50 threads ✅
Inventory pool unaffected → still serving 30 threads ✅
```

---

### Two Bulkhead Implementations

#### Type 1 — Thread Pool Isolation (Heavyweight)

```
Each dependency gets its own thread pool.
Caller thread submits task → returns immediately.
Pool thread does the actual work asynchronously.

  Caller Thread                 Pool Thread
       │                             │
       │  submit(paymentTask) ──────►│
       │◄─ Future returned ──────────│
       │                             │ (doing HTTP call)
       │  (free to do other work)    │
       │                             │ (response arrives)
       │◄─ Future.get() ─────────────│
       │                             │

Pros: Hard isolation — pool exhaustion can't affect caller
Cons: Thread overhead, context switching cost
      Extra latency from handoff between threads
Used by: Hystrix (Netflix original implementation)
```

#### Type 2 — Semaphore Isolation (Lightweight)

```
No separate thread pool.
A semaphore (counter) limits concurrent calls.
Caller thread does the work directly but is blocked
if semaphore count is exhausted.

  Semaphore for NotificationService = 20

  20 concurrent calls in progress:
  semaphore count = 0

  21st call arrives:
  → tryAcquire() fails immediately
  → Request rejected instantly (no waiting)
  → BulkheadFullException thrown

Pros: Minimal overhead, no thread switching
Cons: Uses caller's thread — still occupies it while waiting
      But at least LIMITS how many can pile up
Used by: Resilience4j default bulkhead type
```

---

### Resilience4j Bulkhead — Production Config

**Two modules in Resilience4j:**
- `Bulkhead` → semaphore-based
- `ThreadPoolBulkhead` → thread pool-based

```xml
<!-- pom.xml — already included with resilience4j-spring-boot3 -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

**application.yml — ShopSphere:**
```yaml
resilience4j:
  # Semaphore bulkhead
  bulkhead:
    instances:
      paymentService:
        maxConcurrentCalls: 50        # max 50 concurrent calls
        maxWaitDuration: 10ms         # wait 10ms for a slot
                                      # if full → reject immediately

      notificationService:
        maxConcurrentCalls: 20        # notification gets fewer slots
        maxWaitDuration: 0ms          # don't wait at all → instant reject

      inventoryService:
        maxConcurrentCalls: 30
        maxWaitDuration: 5ms

  # Thread pool bulkhead (async calls)
  thread-pool-bulkhead:
    instances:
      searchService:
        maxThreadPoolSize: 20         # 20 threads for search calls
        coreThreadPoolSize: 10        # keep 10 alive always
        queueCapacity: 50             # queue up to 50 tasks
        keepAliveDuration: 20ms       # idle thread TTL
```

**Service code — semaphore bulkhead:**
```java
@Service
public class OrderService {

    // Semaphore bulkhead — synchronous
    @Bulkhead(name = "paymentService", fallbackMethod = "paymentFallback")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.charge(request);
    }

    // Thread pool bulkhead — async
    @Bulkhead(
        name = "searchService",
        type = Bulkhead.Type.THREADPOOL,   // ← key difference
        fallbackMethod = "searchFallback"
    )
    public CompletableFuture<SearchResponse> searchProducts(String query) {
        return CompletableFuture.supplyAsync(
            () -> searchClient.search(query)
        );
    }

    // Called when bulkhead is full
    private PaymentResponse paymentFallback(
            PaymentRequest request,
            BulkheadFullException ex) {       // ← specific exception type
        log.warn("Payment bulkhead full — {} concurrent calls exceeded",
            "paymentService");
        return PaymentResponse.builder()
            .status(PaymentStatus.PENDING)
            .message("System busy — payment queued")
            .build();
    }

    private PaymentResponse paymentFallback(
            PaymentRequest request,
            Exception ex) {                   // ← catches circuit breaker
        return PaymentResponse.pending(request.getOrderId());
    }
}
```

---

### Bulkhead + Circuit Breaker Together — The Full Shield

These two patterns are **complementary, not alternatives:**

```
Without bulkhead (circuit breaker alone):
  CB protects from cascade — but thread pool still exhausts
  while CB is counting failures to trip.
  Those ~10 failures still consume 10 threads × timeout

With both:
  Bulkhead caps threads instantly
  Circuit breaker detects sustained failure pattern
  Together → zero thread leakage

┌────────────────────────────────────────────────┐
│  Incoming request to Order Service              │
│          │                                     │
│          ▼                                     │
│  ┌───────────────┐  full?  ┌─────────────────┐ │
│  │   Bulkhead    │────────►│    Fallback      │ │
│  │  (20 slots)   │         └─────────────────┘ │
│  └───────┬───────┘                             │
│          │ slot acquired                       │
│          ▼                                     │
│  ┌───────────────┐  open?  ┌─────────────────┐ │
│  │Circuit Breaker│────────►│    Fallback      │ │
│  └───────┬───────┘         └─────────────────┘ │
│          │ circuit closed                      │
│          ▼                                     │
│  ┌───────────────┐ timeout ┌─────────────────┐ │
│  │  TimeLimiter  │────────►│    Fallback      │ │
│  └───────┬───────┘         └─────────────────┘ │
│          │                                     │
│          ▼                                     │
│     Actual HTTP Call                           │
└────────────────────────────────────────────────┘
```

---

### Sizing Your Bulkhead — Little's Law

> **How many concurrent slots should I give each dependency?**

Use **Little's Law:**

```
L = λ × W

L = average number of concurrent requests in system
λ = throughput (requests per second)
W = average latency (seconds)

Example: Payment Service
  λ = 100 req/s hitting Order Service
  W = 200ms average payment call latency = 0.2s
  L = 100 × 0.2 = 20 concurrent calls at any time

  Add 50% safety buffer:
  maxConcurrentCalls = 20 × 1.5 = 30

  If latency spikes to 2s:
  L = 100 × 2 = 200 concurrent → bulkhead kicks in at 30
  → 170 requests rejected fast rather than 200 hanging for 2s
```

```
Quick sizing table for ShopSphere:

  Service         RPS   Latency   L=λW   +50%   Pool Size
  ──────────────────────────────────────────────────────
  Payment         100   200ms     20     30     50 (headroom)
  Inventory       200   50ms      10     15     30
  Notification    50    500ms     25     38     40
  Search (ES)     150   100ms     15     23     25
  Redis Cache     500   5ms       2.5    4      10
```

---

### Observability — Monitoring Bulkhead Health

```java
// Actuator endpoint
GET /actuator/bulkheads

{
  "paymentService": {
    "maxAllowedConcurrentCalls": 50,
    "availableConcurrentCalls": 47,    ← 3 slots in use
    "maxWaitDuration": "10ms"
  },
  "notificationService": {
    "maxAllowedConcurrentCalls": 20,
    "availableConcurrentCalls": 0,     ← FULL! requests being rejected
    "maxWaitDuration": "0ms"
  }
}
```

**Alert when bulkhead is frequently full:**
```java
@EventListener
public void onBulkheadFull(BulkheadOnCallRejectedEvent event) {
    log.error("Bulkhead [{}] full — call rejected. " +
              "Available slots: {}",
        event.getBulkheadName(),
        event.getBulkheadMetrics().getAvailableConcurrentCalls());

    meterRegistry.counter("bulkhead.rejected",
        "name", event.getBulkheadName()).increment();

    // If rejection rate > 10% → page on-call
}
```

---

### Annotation Ordering — The Full Resilience4j Stack

```java
// Correct order (outermost → innermost):

@Bulkhead(name = "paymentService",
          fallbackMethod = "paymentFallback")     // 1st — slot check
@CircuitBreaker(name = "paymentService",
                fallbackMethod = "paymentFallback") // 2nd — health check
@Retry(name = "paymentService")                   // 3rd — retry on fail
@TimeLimiter(name = "paymentService")             // 4th — per-attempt timeout
public CompletableFuture<PaymentResponse> processPayment(...) { }

Execution order when request arrives:
  Bulkhead checks slot availability
    → Circuit Breaker checks if service is healthy
      → Retry wraps the attempt
        → TimeLimiter enforces per-attempt deadline
          → Actual HTTP call
```

---

### ShopSphere Bulkhead Architecture 🛒

```
┌────────────────────────────────────────────────────────┐
│                   Order Service                         │
│                                                         │
│  Incoming Requests (shared HTTP thread pool)            │
│  │                                                      │
│  ├──► [Bulkhead: payment     50 slots] → Payment Svc   │
│  │         CB + Retry on top                           │
│  │                                                      │
│  ├──► [Bulkhead: inventory   30 slots] → Inventory Svc │
│  │         CB + Retry on top                           │
│  │                                                      │
│  ├──► [Bulkhead: notification 20 slots] → Notif Svc    │
│  │         CB only (fire-and-forget, low priority)     │
│  │                                                      │
│  └──► [Bulkhead: search      25 slots] → Search Svc    │
│           ThreadPool type (async search)               │
│                                                         │
│  Notification pool fills up (20/20)?                   │
│  → Notification calls rejected instantly               │
│  → Payment/Inventory/Search completely unaffected ✅   │
└────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What is the bulkhead pattern and what specific failure does it prevent?**
> Bulkhead isolates resources (thread pools or semaphore slots) per downstream dependency. It prevents one slow or failing dependency from exhausting shared thread pool resources and taking down unrelated functionality. Without it, a slow Notification Service can starve threads needed by Payment Service, causing total service failure despite Payment being healthy.

**Q2. What's the difference between semaphore and thread pool bulkhead?**
> Semaphore bulkhead limits concurrent calls using a counter — caller's own thread does the work, but is immediately rejected if the semaphore is exhausted. Thread pool bulkhead uses a dedicated pool — caller submits a task and gets a Future back, providing true async isolation. Thread pool is stronger isolation but carries thread overhead and context-switching cost. Resilience4j defaults to semaphore; use thread pool for genuinely async, latency-sensitive dependencies.

**Q3. How do you size a bulkhead?**
> Use Little's Law: L = λ × W (concurrent calls = throughput × latency). If a service handles 100 req/s and each downstream call takes 200ms, you need 20 concurrent slots at steady state. Add a 50% buffer for spikes. If latency degrades beyond that buffer, the bulkhead rejects excess requests fast rather than letting threads pile up indefinitely.

**Q4. How does bulkhead complement circuit breaker — aren't they solving the same problem?**
> They solve related but distinct problems. Circuit breaker detects failure patterns over time and stops calling a sick service. Bulkhead limits concurrent resource usage right now regardless of health. Without bulkhead, the circuit breaker trips only after N failures — during those N failures, threads are still piling up. With bulkhead, those threads are capped immediately. Together: bulkhead prevents thread exhaustion, circuit breaker prevents sustained overload.

**Q5. A service's bulkhead is frequently full in production. What does that tell you?**
> Either the downstream service is too slow (latency increased, pushing up L = λW), or traffic has grown beyond the original sizing. Investigate: check downstream latency percentiles (p95, p99), check if rejection rate is high, re-apply Little's Law with current numbers. Short term — increase pool size. Long term — fix downstream latency, add more instances, or implement async processing with a queue.

---

### One-Line Summary

> **Bulkhead gives each dependency its own resource pool — when Notification drowns, it drowns alone in its 20-slot pool while Payment and Inventory keep serving normally in theirs. Failure stays contained, not contagious.**

---

Ready for **Topic 8: Timeout & Deadline Propagation** — set timeouts at every layer, propagate deadlines across microservices, and why a missing timeout is a hidden time bomb in your system?

Type **next** 🚀
