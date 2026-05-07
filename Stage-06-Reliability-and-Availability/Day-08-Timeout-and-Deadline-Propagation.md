# Stage 6 — Reliability & Availability
## Topic 8 : Timeout & Deadline Propagation

---

### Why Timeouts Are Non-Negotiable

```
A service with no timeout is a service that can hang forever.

No timeout scenario:
  Order Service calls Payment Service
  Payment Service DB is deadlocked — no response ever

  Order Service thread: waiting...
  t=10s  : still waiting...
  t=60s  : still waiting...
  t=300s : still waiting...
  t=∞    : FOREVER

  Thread never freed.
  100 users hit this → 100 threads gone forever.
  Service dies. Not because of YOUR code — because you
  trusted someone else's service to respond.
```

> **A timeout is not pessimism. It's the contract your service makes with itself: "I will not wait longer than X for anyone."**

---

### Types of Timeouts — Know All Four

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  1. CONNECTION TIMEOUT                              │
│     Time to establish TCP connection                │
│     Service not reachable? Fail fast here.          │
│     Typical: 1–3 seconds                           │
│                                                     │
│  2. READ (SOCKET) TIMEOUT                           │
│     Time to receive data AFTER connection made      │
│     Connected but service hung? Caught here.        │
│     Typical: 2–10 seconds                          │
│                                                     │
│  3. WRITE TIMEOUT                                   │
│     Time to send request data to the server         │
│     Slow upload / large payload? Caught here.       │
│     Less common but important for large POSTs       │
│                                                     │
│  4. CALL / END-TO-END TIMEOUT                       │
│     Total time for entire request lifecycle         │
│     Conn + Write + Wait + Read combined             │
│     Typical: 3–15 seconds                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

```
Timeline of an HTTP call:

  ├──────┤  Connection Timeout (TCP handshake)
         ├───┤  Write Timeout (sending request body)
              ├────────────────┤  Read Timeout (waiting for response)
  ├───────────────────────────┤  End-to-End / Call Timeout
```

---

### The Silent Killer — Missing Timeout Layers

Most developers set ONE timeout. Production needs timeouts at EVERY layer.

```
Request flow through ShopSphere:

  Client (mobile app)
    │  [no timeout set → client hangs forever]
    ▼
  API Gateway
    │  [gateway timeout: 10s]
    ▼
  Order Service (Feign client)
    │  [feign timeout: ???  ← forgot to set this]
    ▼
  Payment Service
    │  [DB query timeout: ???  ← forgot this too]
    ▼
  PostgreSQL

  Payment DB query hangs →
  Payment Service hangs →
  Feign client in Order Service hangs (no timeout!) →
  Order Service thread hangs →
  API Gateway hits its 10s → returns 504 to client →

  BUT: Order Service thread is STILL hanging beyond 10s
       Gateway gave up. Order Service didn't.
       Thread leaked. ← this is the bug.
```

**Every layer needs its own timeout — independently.**

---

### Deadline Propagation — The Core Concept

> **A deadline is a point in absolute time by which the entire operation must complete — propagated across every service in the call chain.**

Timeout vs Deadline:
```
TIMEOUT: "I'll wait 5 seconds from now"
  → Each service gets its own fresh 5s budget
  → Total chain could take 5s × 4 services = 20s
  → Client already gave up at 10s
  → Remaining work is pure waste

DEADLINE: "This entire request must finish by t=10:00:05.000"
  → Absolute timestamp passed through every service
  → Each service knows exactly how much budget remains
  → If budget < estimated work → fail fast immediately
  → Zero wasted processing after client gives up
```

```
With TIMEOUTS only (no deadline):

  Client timeout: 5s
  t=0.0s  Client sends request
  t=0.1s  API Gateway receives, sends to Order Svc (timeout: 5s fresh)
  t=0.5s  Order Svc calls Payment Svc (timeout: 5s fresh)
  t=1.0s  Payment Svc calls DB (timeout: 5s fresh)
  t=5.0s  Client times out → client gets error
  t=6.0s  API Gateway still waiting (its 5s started at t=0.1)
  t=5.5s  Order Svc still processing (its 5s started at t=0.5)
  t=6.0s  Payment Svc still waiting for DB!

  Work continues for 1+ second AFTER client gave up.
  Threads, DB connections, CPU — all wasted.

With DEADLINE propagation:

  Client timeout: 5s → deadline = t + 5s = absolute timestamp
  t=0.0s  deadline = 10:00:05.000 sent in header
  t=0.1s  API Gateway: 4.9s remaining → passes deadline
  t=0.5s  Order Svc: 4.5s remaining → passes deadline
  t=1.0s  Payment Svc: 4.0s remaining → passes deadline
  t=4.9s  Client times out ← error
  t=4.9s  All services check remaining budget = 0.1s
           → All abort immediately ✅
  Zero wasted work.
```

---

### Deadline Propagation in Practice

**Standard mechanism: HTTP Header**

```
grpc-timeout: 4500m          ← gRPC native deadline (milliseconds)
x-request-deadline: 1714567205123  ← Unix timestamp ms (custom)
x-remaining-timeout-ms: 4500      ← remaining budget (simpler)
```

**ShopSphere implementation:**

```java
// DeadlineContext — carries deadline through the call chain
public class DeadlineContext {

    private static final ThreadLocal<Long> deadlineMs =
        new ThreadLocal<>();

    public static void set(long absoluteDeadlineMs) {
        deadlineMs.set(absoluteDeadlineMs);
    }

    public static long get() {
        Long d = deadlineMs.get();
        return d != null ? d : System.currentTimeMillis() + 10_000;
    }

    public static long remainingMs() {
        return get() - System.currentTimeMillis();
    }

    public static boolean isExpired() {
        return remainingMs() <= 0;
    }

    public static void clear() {
        deadlineMs.remove();
    }
}
```

**Incoming request — extract deadline from header:**
```java
// Filter / Interceptor — runs on every incoming request
@Component
public class DeadlineExtractorFilter implements Filter {

    private static final long DEFAULT_TIMEOUT_MS = 10_000;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpReq = (HttpServletRequest) req;
        String deadlineHeader = httpReq.getHeader("x-request-deadline");

        long deadline;
        if (deadlineHeader != null) {
            deadline = Long.parseLong(deadlineHeader);
        } else {
            // First service in chain — set deadline from config
            deadline = System.currentTimeMillis() + DEFAULT_TIMEOUT_MS;
        }

        DeadlineContext.set(deadline);
        try {
            // Check immediately — don't start work on expired request
            if (DeadlineContext.isExpired()) {
                ((HttpServletResponse) res)
                    .sendError(408, "Request deadline already exceeded");
                return;
            }
            chain.doFilter(req, res);
        } finally {
            DeadlineContext.clear();    // prevent thread-local leak
        }
    }
}
```

**Outgoing Feign call — inject deadline into header:**
```java
@Component
public class DeadlinePropagationInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        long remaining = DeadlineContext.remainingMs();

        // Propagate absolute deadline to downstream
        template.header("x-request-deadline",
            String.valueOf(DeadlineContext.get()));

        // Also propagate remaining ms (convenience for logging)
        template.header("x-remaining-timeout-ms",
            String.valueOf(remaining));

        // Dynamically set Feign read timeout based on remaining budget
        // Leave 20% as safety margin for response processing
        long feignTimeout = (long)(remaining * 0.8);
        if (feignTimeout < 100) {
            // Less than 100ms left — don't even try
            throw new DeadlineExceededException(
                "Insufficient budget to make downstream call: "
                + remaining + "ms remaining");
        }
    }
}
```

**Before expensive operations — check budget:**
```java
@Service
public class OrderService {

    public OrderResponse createOrder(CreateOrderRequest request) {

        // Check 1: Before DB write
        if (DeadlineContext.remainingMs() < 500) {
            throw new DeadlineExceededException("Insufficient budget for DB write");
        }
        orderRepository.save(order);

        // Check 2: Before calling Payment
        if (DeadlineContext.remainingMs() < 2000) {
            throw new DeadlineExceededException("Insufficient budget for payment");
        }
        paymentService.charge(request);

        // Check 3: Before calling Notification (optional — low priority)
        if (DeadlineContext.remainingMs() > 500) {
            notificationService.sendConfirmation(order);
        } else {
            // Not enough budget — publish to Kafka, handle async
            kafkaTemplate.send("order.confirmed", order.getId());
        }

        return OrderResponse.from(order);
    }
}
```

---

### Configuring Timeouts in ShopSphere Stack

**Feign Client timeouts:**
```yaml
# application.yml
spring:
  cloud:
    openfeign:
      client:
        config:
          # Default for all Feign clients
          default:
            connectTimeout: 1000     # 1s to connect
            readTimeout: 5000        # 5s to read response

          # Per-service overrides
          paymentService:
            connectTimeout: 1000
            readTimeout: 3000        # payment must be fast

          notificationService:
            connectTimeout: 1000
            readTimeout: 8000        # notification can be slower
```

**HikariCP DB connection timeouts:**
```yaml
spring:
  datasource:
    hikari:
      connectionTimeout: 3000       # 3s to get connection from pool
      idleTimeout: 600000           # 10min idle before eviction
      maxLifetime: 1800000          # 30min max connection age
      keepaliveTime: 30000          # 30s keepalive ping

# Per-query timeout in JPA
spring:
  jpa:
    properties:
      jakarta:
        persistence:
          query:
            timeout: 5000           # 5s per query
```

**Redis timeouts:**
```yaml
spring:
  data:
    redis:
      timeout: 500ms                # 500ms command timeout
      connect-timeout: 1000ms       # 1s connection timeout
      lettuce:
        pool:
          max-wait: 200ms           # 200ms to get pool connection
```

**API Gateway (Spring Cloud Gateway) timeouts:**
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000       # 1s
        response-timeout: 10s       # 10s total

      # Per-route overrides
      routes:
        - id: payment-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          metadata:
            response-timeout: 5000  # payment gets 5s
            connect-timeout: 1000

        - id: search-route
          uri: lb://search-service
          predicates:
            - Path=/api/search/**
          metadata:
            response-timeout: 3000  # search gets 3s
```

---

### gRPC Deadlines — The Gold Standard

gRPC has deadline propagation built into the protocol natively.

```java
// gRPC client — set deadline once, propagated automatically
PaymentResponse response = paymentStub
    .withDeadlineAfter(3, TimeUnit.SECONDS)  // deadline set here
    .charge(request);

// gRPC server — check deadline inside handler
@Override
public void charge(PaymentRequest request,
                   StreamObserver<PaymentResponse> responseObserver) {

    // Check if caller already gave up
    if (Context.current().getDeadline() != null &&
        Context.current().getDeadline().isExpired()) {
        responseObserver.onError(
            Status.DEADLINE_EXCEEDED.asRuntimeException());
        return;
    }

    // Do work...
    // gRPC propagates deadline to any further calls automatically
}
```

> gRPC is why Google's internal systems (Borgmaster, Spanner) handle deadline propagation correctly at scale. HTTP/REST needs manual header passing — gRPC bakes it in.

---

### Timeout Budgeting — The Full Picture

```
Client gives: 10 seconds total

API Gateway:        uses 0.1s  → passes 9.9s deadline downstream
Order Service:      uses 0.5s  → 9.4s remaining
  DB write:         uses 0.3s  → 9.1s remaining
  Payment call:     uses 1.0s  → 8.1s remaining  ← 3s budget given
  Inventory call:   uses 0.2s  → 7.9s remaining  ← 2s budget given
  Notification:     uses 0.1s  → 7.8s remaining  (async via Kafka)

Total used: 2.2s out of 10s budget.

Under degraded conditions (Payment slow):
  Payment call:     uses 2.9s  ← close to 3s budget
  Remaining:        5.3s — still enough for rest of chain ✅

Under severe degradation (Payment at 8s):
  Order Service has 9.4s - 0.3s(DB) = 9.1s before Payment call
  Payment call budget = 3s (capped)
  Payment times out at 3s → fast fail
  Order Service continues → fallback to PENDING payment ✅
  Total: ~3.5s — client gets response before 10s deadline ✅
```

---

### The Timeout Anti-Patterns

```
❌ Anti-pattern 1: Same timeout everywhere
   connectTimeout = readTimeout = callTimeout = 30s
   → Everything hangs for 30s before failing
   → Thread pool exhaustion in 200/30 = 6 req/s

❌ Anti-pattern 2: No timeout on DB queries
   ORM default = no timeout
   One bad query can hold a connection forever
   Fix: Always set query timeout + HikariCP connectionTimeout

❌ Anti-pattern 3: Timeout > upstream timeout
   Gateway timeout: 5s
   Order Service Feign timeout: 10s
   → Gateway gives up at 5s
   → Order Service keeps waiting until 10s
   → Thread leaked for 5 extra seconds

   Fix: Inner timeouts must be SHORTER than outer timeouts
        Gateway: 10s > OrderSvc: 8s > PaymentSvc: 3s

❌ Anti-pattern 4: Retry + high timeout = disaster
   Timeout: 10s, Retries: 3
   → Single call can take 30s (3 × 10s)
   → 200 threads / 30s = 6 req/s before exhaustion
   Fix: timeout × maxAttempts must fit within outer deadline
```

---

### ShopSphere Timeout Budget Map 🛒

```
┌──────────────────────────────────────────────────────────┐
│              ShopSphere Timeout Hierarchy                 │
│                                                          │
│  Mobile Client          ──── 15s total deadline          │
│       │                                                  │
│  API Gateway            ──── 12s timeout                 │
│       │                                                  │
│  Order Service          ──── 10s budget                  │
│  ├── DB write           ──── 2s query timeout            │
│  ├── Inventory Feign    ──── 1s read timeout (GET)       │
│  ├── Payment Feign      ──── 3s read timeout             │
│  │      └── Payment DB  ──── 2s query timeout            │
│  └── Notification       ──── 500ms (or Kafka async)      │
│                                                          │
│  Each downstream budget < parent budget                  │
│  Sum of all children < parent timeout ✅                 │
│                                                          │
│  Deadline header: x-request-deadline propagated          │
│  through every Feign call automatically via              │
│  DeadlinePropagationInterceptor                          │
└──────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What's the difference between a timeout and a deadline?**
> A timeout is relative — "wait 5 seconds from now." Each service in a chain gets a fresh timeout, so the total chain can far exceed what the client allows. A deadline is absolute — a timestamp by which everything must complete. It propagates across all services, so each one knows exactly how much budget remains. If budget is insufficient, services fail fast immediately rather than wasting resources on work the client already gave up on.

**Q2. A request comes in with 10ms left on its deadline. What should your service do?**
> Reject it immediately with a 408 or deadline exceeded error. Starting work with 10ms left is wasteful — DB round trips alone take 1–5ms, and any downstream call will certainly exceed the budget. Failing fast frees the thread immediately for requests that have a realistic chance of completing.

**Q3. Why must inner timeouts always be shorter than outer timeouts?**
> If the inner timeout is longer than the outer, the outer layer (e.g., gateway) gives up and returns an error to the client, but the inner layer's thread keeps running until its longer timeout fires — leaking threads. The rule is: leaf service timeout < intermediate service timeout < gateway timeout < client timeout. This ensures when the outer layer gives up, inner layers also give up, freeing all resources simultaneously.

**Q4. How do you propagate deadlines across microservices in REST?**
> Use a custom HTTP header like `x-request-deadline` carrying the absolute Unix timestamp. On the incoming side, a servlet filter extracts the header and stores it in a ThreadLocal. On the outgoing side, a Feign RequestInterceptor reads the ThreadLocal and injects the header into every downstream call. Services check remaining budget before starting expensive operations and abort early if budget is insufficient.

**Q5. What timeout configuration is most commonly forgotten?**
> DB query timeout. Most teams set Feign and HTTP timeouts but forget that JPA/Hibernate has no query timeout by default. A single runaway query can hold a DB connection indefinitely, exhausting the HikariCP pool silently. Always set `jakarta.persistence.query.timeout` in JPA properties and `connectionTimeout` in HikariCP config.

---

### One-Line Summary

> **Set timeouts at every layer — connection, read, query, gateway — with inner timeouts always shorter than outer. Propagate an absolute deadline across all services so everyone fails fast together the moment the client gives up, wasting zero threads on orphaned work.**

---

Ready for **Topic 9: SLA / SLO / SLI** — how uptime is contractually defined, measured, and monitored in production, error budgets, and what happens when you breach your SLO?

Type **next** 🚀
