## Topic 6 — Distributed Tracing

### The Problem First

In a monolith, debugging a slow request is straightforward:

```
User reports: "Checkout is slow"

You check one log file:
  [INFO] OrderController.placeOrder() - started
  [INFO] PaymentService.charge() - started  
  [INFO] PaymentService.charge() - took 2300ms  ← found it
  [INFO] OrderController.placeOrder() - done
```

In microservices, that same request touches many services:

```
User hits "Place Order"
  → API Gateway
    → Order Service
      → Product Service (check stock)
      → Payment Service
        → Fraud Detection Service
      → Notification Service
        → Email Provider

Request is slow. Which service caused it?
Each service has its OWN log file.
Logs from 6 services are interleaved with thousands of
other requests happening simultaneously.

Without tracing: you're blind.
```

---

### What is Distributed Tracing?

A system that **tracks a single request as it flows across multiple services**, recording timing and metadata at each hop.

Every request gets a unique **Trace ID** that propagates through every service it touches. Each unit of work within a service creates a **Span**.

```
Trace = the entire journey of one request end-to-end
Span  = one unit of work within that journey (one service call, one DB query)

A Trace is made up of many nested Spans.
```

---

### Core Concepts

#### Trace ID

A globally unique identifier assigned at the entry point (API Gateway or first service). Propagated in HTTP headers to every downstream service.

```
Request arrives at API Gateway
  → Gateway generates: traceId = "abc123xyz"
  → Every downstream HTTP call adds header:
      X-B3-TraceId: abc123xyz   (Zipkin format)
      traceparent: 00-abc123xyz-spanId-01  (W3C format)
  → Every service that receives this header logs it
  → All logs for this request are tagged with "abc123xyz"
```

Now you can search logs across ALL services:
```
grep "abc123xyz" service-logs/* → complete picture of one request
```

#### Span

A named, timed operation. Spans form a **tree** — parent spans contain child spans.

```
Span structure:
  {
    traceId:   "abc123xyz",
    spanId:    "span-001",
    parentId:  null,              ← root span (no parent)
    name:      "POST /orders",
    service:   "order-service",
    startTime: 1700000000000,
    duration:  340ms,
    tags: {
      http.status: 200,
      db.type: postgresql
    }
  }
```

#### Span Relationships — The Trace Tree

```
Trace: abc123xyz  (total: 340ms)
│
└── [order-service] POST /orders  (340ms)  ← root span
      │
      ├── [order-service] SELECT orders DB  (12ms)
      │
      ├── [product-service] GET /products/stock  (45ms)
      │     └── [product-service] SELECT products DB  (8ms)
      │
      ├── [payment-service] POST /payments  (260ms)  ← SLOW
      │     ├── [payment-service] fraud-check RPC  (230ms)  ← ROOT CAUSE
      │     └── [payment-service] INSERT payments DB  (15ms)
      │
      └── [notification-service] POST /notify  (18ms)
            └── [notification-service] publish RabbitMQ  (5ms)
```

One glance at this tree tells you:
- Total request took 340ms
- Payment Service took 260ms of that
- Inside Payment, fraud-check took 230ms — that's your bottleneck

**This is impossible to see from raw logs alone.**

---

### How Trace Context Propagates

```
                    HTTP Headers carry the trace context

Order Service                    Product Service
─────────────────────────────    ─────────────────────────
Receives request                 Receives request from Order
Creates root span                Extracts traceId from header
                                 Creates child span with same traceId

FeignClient call adds headers:
  GET /products/stock
  X-B3-TraceId: abc123xyz  ──────────────────────────────────▶
  X-B3-SpanId:  span-003
  X-B3-ParentSpanId: span-001

                                 Product Service logs:
                                 traceId=abc123xyz spanId=span-004
                                 parentSpanId=span-003
```

The **propagation** is automatic when using instrumentation libraries — you don't manually set these headers.

---

### The Two Major Tools

#### Jaeger

Open-source distributed tracing by Uber, now a CNCF project. Production-grade, widely used with Kubernetes.

```
Jaeger Architecture:

  Your Services (instrumented with OpenTelemetry)
       │ emit spans via UDP/HTTP
       ▼
  Jaeger Collector
       │ validates, processes
       ▼
  Storage (Elasticsearch / Cassandra)
       │
       ▼
  Jaeger Query + UI
       │
       ▼
  You search: traceId=abc123 → see full trace tree visually
```

Jaeger UI shows the **Gantt chart** of spans — you instantly see which service took how long.

#### Zipkin

Open-source by Twitter. Older than Jaeger, simpler. Same concept: collect spans, store them, provide a search UI.

```
Key difference:
  Zipkin → simpler, less features, easier to self-host
  Jaeger → more scalable, better UI, CNCF standard
  
In modern stacks: Jaeger is preferred.
Both speak OpenTelemetry format now.
```

---

### OpenTelemetry — The Standard

Before OpenTelemetry, every tracing tool had its own SDK:
- Zipkin had its Brave library
- Jaeger had its own client
- AWS X-Ray had its own SDK

Switching tools meant rewriting instrumentation code.

**OpenTelemetry (OTel)** is the vendor-neutral standard:

```
Your Service
  │
  │ uses OpenTelemetry SDK (one library)
  │
  ▼
OTel Collector
  │
  ├──▶ Jaeger
  ├──▶ Zipkin
  ├──▶ AWS X-Ray
  └──▶ Datadog

You instrument ONCE. Switch backends without code changes.
```

In Spring Boot 3.x, this is built in via **Micrometer Tracing**:

```xml
<!-- Spring Boot 3.x — automatic trace propagation -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0   # trace 100% of requests (use 0.1 in prod = 10%)
  zipkin:
    tracing:
      endpoint: http://jaeger:9411/api/v2/spans
```

With this config, Spring Boot **automatically**:
- Creates spans for every incoming HTTP request
- Creates child spans for every outgoing Feign/RestTemplate call
- Propagates traceId through all HTTP headers
- Creates spans for DB queries (if using Spring Data)
- Exports everything to Jaeger/Zipkin

Zero manual span creation needed for basic tracing.

---

### Correlation IDs vs Trace IDs

These are related but distinct:

```
Correlation ID:
  - Business-level identifier
  - Manually set, manually propagated
  - Example: "order-request-uuid" you generate at order creation
  - Stored in DB, referenced in support tickets
  - Survives async boundaries (stored in Kafka message headers)

Trace ID:
  - Infrastructure-level identifier  
  - Auto-generated by tracing library
  - Lives only for the duration of a synchronous request chain
  - NOT automatically propagated through Kafka/RabbitMQ
    (needs manual instrumentation for async)

Best practice: use BOTH
  - traceId for synchronous request debugging
  - correlationId for end-to-end business flow tracking
    across sync + async boundaries
```

---

### Sampling — You Can't Trace Everything

In production, 100% trace collection is expensive:

```
10,000 requests/second
Each request touches 8 services = 80,000 spans/second
Each span = ~1KB → 80MB/second of trace data
→ Elasticsearch fills up fast, high overhead

Solution: Sampling
```

#### Head-Based Sampling
Decision made at the **start** of the request:

```
probability: 0.01  → trace 1% of all requests randomly
                   → 100 requests/second traced instead of 10,000

Problem: you might miss the 1 slow request that happens
         to not be in the sampled 1%
```

#### Tail-Based Sampling
Decision made **after** the request completes:

```
Collect ALL spans temporarily in a buffer.
After request completes, evaluate:
  - Duration > 2 seconds? → KEEP (always interesting)
  - HTTP 5xx? → KEEP (errors always interesting)
  - Normal fast request? → DISCARD (not interesting)

Pro: never miss errors or slow requests
Con: needs a buffering layer (OTel Collector handles this)
```

**Production approach**: tail-based sampling — always capture errors and slow requests, sample normal traffic.

---

### Manual Span Creation

For important business operations not covered by auto-instrumentation:

```java
@Service
public class OrderService {

    private final Tracer tracer; // injected by Spring/OTel
    
    public Order placeOrder(OrderRequest request) {
        
        // Create a custom span for fraud check business logic
        Span fraudCheckSpan = tracer.nextSpan()
            .name("fraud-check")
            .tag("userId", request.getUserId())
            .tag("amount", String.valueOf(request.getAmount()))
            .start();
        
        try (Tracer.SpanInScope scope = tracer.withSpan(fraudCheckSpan)) {
            boolean isFraud = fraudService.check(request);
            fraudCheckSpan.tag("result", isFraud ? "flagged" : "clean");
            return processOrder(request);
        } catch (Exception e) {
            fraudCheckSpan.error(e);
            throw e;
        } finally {
            fraudCheckSpan.end(); // always end spans
        }
    }
}
```

Now your Jaeger trace tree shows `fraud-check` as a named span with duration and tags.

---

### Tracing Across Async Boundaries

Kafka/RabbitMQ breaks automatic propagation — traceId doesn't travel in message headers by default:

```
Order Service publishes Kafka event
  → traceId lives in HTTP thread context
  → Kafka message is fire-and-forget
  → Notification Service consumes later in a NEW thread
  → New thread has NO traceId → trace chain broken

Fix: manually propagate trace context in message headers

// Producer side (Order Service)
Map<String, String> traceHeaders = new HashMap<>();
tracer.currentTraceContext().injector(Map::put)
      .inject(tracer.currentTraceContext().context(), traceHeaders);
// Add traceHeaders to Kafka ProducerRecord headers

// Consumer side (Notification Service)  
// Extract trace context from Kafka headers
// Create child span with parent = extracted context
```

Spring Kafka + OTel auto-instrumentation handles this automatically if configured correctly.

---

### ShopSphere Lens

```
ShopSphere already has Micrometer + Actuator (Topic 7 of your 
Spring Boot curriculum). Distributed tracing is the natural next layer.

Current gap in ShopSphere:
  Services log with MDC (manual correlationId) but no
  automatic cross-service trace propagation.

What to add:
  1. micrometer-tracing-bridge-otel to each service's pom.xml
  2. Jaeger as a Docker Compose service
  3. management.tracing.sampling.probability=1.0 in dev

Then a single order request shows:
  api-gateway (5ms)
  └── order-service (340ms)
        ├── DB insert (12ms)
        ├── product-service stock check (45ms)  
        ├── kafka publish (8ms)
        └── feign → notification-service (18ms)

One URL in Jaeger UI → full picture.
No grep-across-log-files needed.
```

---

### Interview Questions

**Q1. What is distributed tracing and why can't you just use centralized logging?**

> Centralized logging aggregates logs but logs from different services for the same request are interleaved with thousands of other requests. Without a shared identifier propagated across services, you can't reconstruct the journey of a single request. Distributed tracing assigns a unique traceId to each request, propagates it through all service calls, and records timing at each hop — giving you a visual Gantt chart of exactly where time was spent.

**Q2. What's the difference between a Trace and a Span?**

> A Trace represents the entire end-to-end journey of one request across all services. A Span is a single named, timed operation within that journey — one service handling a request, one DB query, one downstream call. A Trace is a tree of Spans, where parent-child relationships show the call hierarchy.

**Q3. What is OpenTelemetry and why does it matter?**

> OpenTelemetry is a vendor-neutral standard for distributed tracing, metrics, and logging instrumentation. Before it, every tracing backend (Jaeger, Zipkin, Datadog) had its own SDK — switching backends meant rewriting instrumentation. OTel lets you instrument once and route telemetry to any backend via the OTel Collector. Spring Boot 3.x has first-class OTel support via Micrometer Tracing.

**Q4. Why doesn't traceId automatically propagate through Kafka?**

> HTTP-based tracing works because the instrumentation library intercepts outgoing HTTP calls and injects trace headers. Kafka is asynchronous — the message is decoupled from the producing thread's context. The consumer runs in a completely different thread, possibly later, with no HTTP headers. You must explicitly serialize the trace context into Kafka message headers on the producer side and deserialize it on the consumer side to maintain trace continuity across async boundaries.

---

Ready for **Topic 7 — Service Mesh** when you are.
