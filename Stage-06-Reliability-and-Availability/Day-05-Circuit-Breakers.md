# Stage 6 — Reliability & Availability
## Topic 5 : Circuit Breakers

---

### Why Circuit Breakers Exist

Without a circuit breaker, one slow service can kill your entire system.

```
Normal flow:
  Order Service → Payment Service (responds in 50ms) ✅

Payment Service starts struggling (GC pause / DB overload):
  Order Service → Payment Service (no response...)
                                  (waiting...)
                                  (30s timeout...)
                                  (finally: error)

Meanwhile:
  100 users/sec hitting Order Service
  Each request holds a thread for 30 seconds
  100 req/s × 30s = 3,000 threads stuck waiting

  Order Service thread pool = 200 threads
  All 200 exhausted in 2 seconds
  Order Service is now DOWN too 💀

  Now API Gateway can't reach Order Service...
  API Gateway threads pile up...
  API Gateway is DOWN too 💀

This is called CASCADE FAILURE — one sick service
takes down the entire system like dominoes.
```

**Circuit Breaker breaks the cascade:**
```
Payment Service struggling →
Circuit Breaker OPENS →
Order Service STOPS calling Payment →
Order Service returns fast error immediately →
Threads freed instantly →
Order Service stays healthy ✅
Payment Service gets breathing room to recover ✅
```

---

### The State Machine — Core of Circuit Breaker

A circuit breaker is literally a **3-state finite state machine.**

```
                    failure threshold
                       exceeded
         ┌─────────────────────────────────┐
         │                                 ▼
    ┌────┴────┐                       ┌─────────┐
    │ CLOSED  │                       │  OPEN   │
    │(normal) │                       │(blocked)│
    └─────────┘                       └────┬────┘
         ▲                                 │
         │                         reset timeout
         │                           expires
         │                                 │
         │       ┌──────────────┐          │
         │       │  HALF-OPEN   │◄─────────┘
         └───────│  (testing)   │
    success      └──────────────┘
    threshold          │
    reached            │ failure occurs
                       │
                       ▼
                   back to OPEN
```

---

### State 1 — CLOSED (Normal Operation)

```
All requests flow through normally.
Circuit breaker is counting failures in a sliding window.

  Request 1  → Service B → ✅ success  (fail count: 0)
  Request 2  → Service B → ✅ success  (fail count: 0)
  Request 3  → Service B → ❌ failure  (fail count: 1)
  Request 4  → Service B → ❌ failure  (fail count: 2)
  Request 5  → Service B → ❌ failure  (fail count: 3)

  Failure rate = 3/5 = 60%
  Threshold = 50%

  → TRIP! Circuit opens. ⚡
```

**Sliding window types:**

```
COUNT-BASED window (last N calls):
  Window size = 10
  If 6 of last 10 calls failed → trip

  [✅][✅][❌][❌][❌][❌][❌][❌][✅][❌]
         ↑ oldest                  ↑ newest
  Failures = 8/10 = 80% → OPEN

TIME-BASED window (last N seconds):
  Window = 60 seconds
  If failure rate > 50% in last 60s → trip

  More representative under variable traffic
  Resilience4j supports both
```

---

### State 2 — OPEN (Blocking All Requests)

```
Circuit is OPEN = breaker is tripped = wall is up.

  Request 6  → Circuit Breaker → ❌ FAST FAIL (no call made)
  Request 7  → Circuit Breaker → ❌ FAST FAIL
  Request 8  → Circuit Breaker → ❌ FAST FAIL

  Service B is NOT called at all.
  Responses return in microseconds (not 30s timeouts).
  Thread pool is freed.

  A reset timer starts: e.g. 30 seconds
  After 30s → move to HALF-OPEN to test recovery
```

**The fallback runs here:**
```java
// Instead of error, return a safe default
return getFallbackResponse();
// e.g. cached data, default value, or graceful degradation
```

---

### State 3 — HALF-OPEN (Testing Recovery)

```
Reset timer expired. "Is the service healthy again?"
Let through a LIMITED number of probe requests.

  Request 9  → Service B → ✅ success  (1/3 probes OK)
  Request 10 → Service B → ✅ success  (2/3 probes OK)
  Request 11 → Service B → ✅ success  (3/3 probes OK)

  All probes passed → CLOSE circuit → back to normal ✅

  OR:

  Request 9  → Service B → ❌ failure
  → Immediately back to OPEN, reset timer restarts
  → Service still sick, don't hammer it
```

```
HALF-OPEN is critical:
  Without it → circuit stays open forever even after recovery
  With it    → system self-heals automatically
```

---

### Resilience4j — Production Implementation

Resilience4j is the standard circuit breaker library for Spring Boot (replaced Hystrix which is dead).

**Dependency (ShopSphere pom.xml):**
```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<!-- also needs spring-boot-starter-aop -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

**application.yml — ShopSphere Order → Payment circuit:**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # Sliding window
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10           # last 10 calls
        failureRateThreshold: 50        # >50% fail → OPEN
        slowCallRateThreshold: 80       # >80% slow → OPEN too
        slowCallDurationThreshold: 2000 # >2s = "slow call"

        # OPEN → HALF-OPEN
        waitDurationInOpenState: 30s    # wait 30s before probing

        # HALF-OPEN probing
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true

        # Minimum calls before calculating failure rate
        minimumNumberOfCalls: 5

        # What counts as failure
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - feign.FeignException
        ignoreExceptions:
          - com.shopsphere.exception.ValidationException
```

**Service code:**
```java
@Service
public class OrderService {

    private final PaymentClient paymentClient;

    // ✅ Circuit breaker wraps the call
    // ✅ Fallback method called when circuit is OPEN
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.charge(request);  // Feign call
    }

    // Fallback — called when circuit is OPEN or call fails
    private PaymentResponse paymentFallback(PaymentRequest request,
                                             Exception ex) {
        log.warn("Payment service unavailable, using fallback. Error: {}",
                  ex.getMessage());

        // Option A: Return a pending state — retry asynchronously
        return PaymentResponse.builder()
            .status(PaymentStatus.PENDING)
            .message("Payment queued — will be processed shortly")
            .build();

        // Option B: Throw a specific exception for caller to handle
        // throw new PaymentServiceUnavailableException("...");
    }
}
```

---

### Combining Circuit Breaker with Other Patterns

Circuit breaker works best layered with retry and timeout:

```
Request
   │
   ▼
┌──────────┐     Circuit OPEN?
│ Circuit  │────────────────────► Fallback (fast)
│ Breaker  │
└────┬─────┘
     │ Circuit CLOSED/HALF-OPEN
     ▼
┌──────────┐     Timeout exceeded?
│ Timeout  │────────────────────► TimeoutException → CB records failure
└────┬─────┘
     │
     ▼
┌──────────┐     Failed? Retry up to N times
│  Retry   │────────────────────► Each failure recorded by CB
└────┬─────┘
     │
     ▼
  Actual Service Call
```

**IMPORTANT ordering rule:**
```
Retry sits INSIDE Circuit Breaker.

Why: If CB wraps Retry → 3 retries = 3 failures recorded → CB trips faster
     If Retry wraps CB → retries happen after CB already blocked → wasted

Resilience4j annotation order (outermost first = applied last):
  @CircuitBreaker   ← outermost — decides to call or block
  @Retry            ← middle — retries on failure
  @TimeLimiter      ← innermost — enforces timeout per attempt
```

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
@Retry(name = "paymentService")
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResponse> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
}
```

---

### Monitoring Circuit Breaker State

Resilience4j exposes state via Spring Actuator:

```bash
GET /actuator/circuitbreakers

{
  "paymentService": {
    "state": "CLOSED",
    "failureRate": "20.0%",
    "slowCallRate": "0.0%",
    "bufferedCalls": 10,
    "failedCalls": 2,
    "successfulCalls": 8,
    "notPermittedCalls": 0   ← calls blocked when OPEN
  }
}
```

**Alerting on state transitions:**
```java
@EventListener
public void onCircuitBreakerStateChange(CircuitBreakerOnStateTransitionEvent event) {
    log.error("Circuit breaker [{}] transitioned from {} to {}",
        event.getCircuitBreakerName(),
        event.getStateTransition().getFromState(),
        event.getStateTransition().getToState()
    );
    // Fire PagerDuty / Slack alert here
    alertingService.sendAlert("Circuit breaker OPEN: " +
        event.getCircuitBreakerName());
}
```

---

### ShopSphere Circuit Breaker Map 🛒

```
┌─────────────────────────────────────────────────────────┐
│              ShopSphere Circuit Breaker Topology         │
│                                                          │
│  API Gateway                                             │
│       │                                                  │
│  ┌────▼──────────────────────────────────────────────┐  │
│  │  Order Service                                     │  │
│  │                                                    │  │
│  │  [CB: inventoryService] → Inventory Service        │  │
│  │        fallback: use cached stock, flag for review  │  │
│  │                                                    │  │
│  │  [CB: paymentService]   → Payment Service          │  │
│  │        fallback: queue payment, return PENDING     │  │
│  │                                                    │  │
│  │  [CB: notificationService] → Notification Service  │  │
│  │        fallback: log + retry via Kafka queue       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Search Service                                          │
│  [CB: elasticsearchService] → Elasticsearch             │
│        fallback: DB query (slower but functional)        │
│                                                          │
│  Product Service                                         │
│  [CB: redisCache] → Redis                               │
│        fallback: skip cache, hit DB directly            │
└─────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What problem does a circuit breaker solve that a timeout alone doesn't?**
> A timeout prevents a single request from hanging forever, but if the downstream service is consistently failing, every request still waits for the full timeout before failing. With 200 threads and a 30s timeout, your thread pool exhausts in 2 seconds. A circuit breaker detects the pattern and stops making calls entirely, returning fast failures immediately and protecting thread pool resources.

**Q2. Walk me through the three states of a circuit breaker.**
> CLOSED is normal operation — requests flow through and failures are counted. When failure rate exceeds threshold, it trips to OPEN — all requests fast-fail without calling the service, giving it recovery time. After a reset timeout, it moves to HALF-OPEN — a limited number of probe requests are allowed through. If probes succeed, it returns to CLOSED. If any probe fails, it goes back to OPEN.

**Q3. What's the difference between a circuit breaker and a retry?**
> Retry handles transient failures — temporary blips where retrying immediately makes sense. Circuit breaker handles sustained failures — when a service is genuinely down, retrying repeatedly makes things worse by hammering a struggling service. They're complementary: retry handles the occasional hiccup, circuit breaker handles prolonged outages.

**Q4. What should a good fallback do?**
> It depends on criticality. For non-critical paths (notifications, recommendations) — return a cached or default response silently. For semi-critical paths (search) — degrade gracefully (hit DB instead of ES). For critical paths (payment) — return a PENDING state and queue for async retry. Never silently swallow errors for critical operations.

**Q5. How does Resilience4j differ from the old Netflix Hystrix?**
> Hystrix is deprecated and no longer maintained. Resilience4j is lightweight, functional, designed for Java 8+, and doesn't require a separate thread pool per command (uses the caller's thread by default). It also integrates natively with Spring Boot 3 Actuator for metrics and supports more patterns — bulkhead, rate limiter, time limiter — all composable via annotations.

---

### One-Line Summary

> **Circuit breaker is a 3-state machine — CLOSED (normal) → OPEN (block all calls, fast fail) → HALF-OPEN (probe recovery) — it stops cascade failures by refusing to call a sick service, giving it time to recover while keeping your service healthy.**

---

Ready for **Topic 6: Retry Logic with Exponential Backoff** — how to retry intelligently, what jitter is and why you need it, and why naive retries can kill a recovering service?

Type **next** 🚀
