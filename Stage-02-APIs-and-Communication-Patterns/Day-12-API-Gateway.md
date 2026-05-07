# Stage 2 — Topic 12: API Gateway

## Theory

We have referenced the API Gateway throughout every topic in Stage 2. Now we go deep — the internals, the patterns, the failure modes, and exactly how ShopSphere's gateway is architected.

An API Gateway is the **single entry point** for all client traffic into your microservices system. Every request from every client — mobile app, web frontend, third-party developer, internal service — passes through it before reaching any downstream service.

**Why it exists — the without-it problem:**

```
Without API Gateway:
  Mobile App knows about:
    - Order Service:        order.shopsphere.com:8081
    - Product Service:      product.shopsphere.com:8082
    - User Service:         user.shopsphere.com:8083
    - Search Service:       search.shopsphere.com:8084
    - Payment Service:      payment.shopsphere.com:8085
    - ... 22 services

  Problems:
    Client must know every service's address
    Every service implements auth, rate limiting, CORS, SSL — duplicated 22 times
    Changing internal service layout requires updating every client
    No single place to enforce security policies
    CORS nightmare — 22 different origins
    No unified logging or tracing entry point
    Mobile app cannot be updated instantly — old clients break

With API Gateway:
  Mobile App knows about:
    - api.shopsphere.com (one address, forever stable)

  Gateway handles everything else internally.
```

---

## What an API Gateway Does — The Full Responsibility List

```
Incoming Request
      ↓
┌─────────────────────────────────────────────────────┐
│                    API Gateway                       │
│                                                      │
│  1.  SSL Termination        decrypt HTTPS            │
│  2.  Request Logging        log every request        │
│  3.  Rate Limiting          enforce quotas           │
│  4.  Authentication         verify JWT               │
│  5.  Authorization          check role/path access   │
│  6.  Request Validation     reject malformed input   │
│  7.  CORS                   handle preflight         │
│  8.  Request Transformation add/remove headers       │
│  9.  Routing                forward to correct svc   │
│  10. Load Balancing         distribute across instances│
│  11. Circuit Breaking       fail fast on down svc    │
│  12. Retry Logic            retry on transient failures│
│  13. Response Transformation modify response if needed│
│  14. Response Logging       log status + latency     │
│  15. Response Compression   gzip responses           │
└─────────────────────────────────────────────────────┘
      ↓
Downstream Microservices
```

None of these are business logic — they are infrastructure concerns. The gateway centralises them so every microservice gets them for free.

---

## Internals — Request Routing

Routing is the core function. The gateway matches incoming requests to downstream services using rules:

### Path-Based Routing

```yaml
# Spring Cloud Gateway routing configuration

spring:
  cloud:
    gateway:
      routes:

        # Order Service
        - id: order-service
          uri: lb://order-service          # lb:// = use service discovery
          predicates:
            - Path=/api/v1/orders/**       # match this path pattern
          filters:
            - StripPrefix=2                # strip /api/v1 before forwarding

        # Product Service
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/v1/products/**
          filters:
            - StripPrefix=2

        # Search Service
        - id: search-service
          uri: lb://search-service
          predicates:
            - Path=/api/v1/search/**
          filters:
            - StripPrefix=2

        # Admin routes — additional role check
        - id: admin-routes
          uri: lb://admin-service
          predicates:
            - Path=/api/v1/admin/**
          filters:
            - StripPrefix=2
            - name: CircuitBreaker
              args:
                name: adminCircuitBreaker
                fallbackUri: forward:/fallback/admin
```

**How routing resolves:**
```
Incoming: GET https://api.shopsphere.com/api/v1/orders/o-789

Gateway:
  1. Match path /api/v1/orders/** → order-service route
  2. StripPrefix=2 → remove /api/v1 → /orders/o-789
  3. Resolve lb://order-service via service discovery → 10.0.1.5:8081
  4. Forward: GET http://10.0.1.5:8081/orders/o-789
  5. Return response to client
```

### Header-Based Routing

```yaml
# Route different API versions to different service instances
- id: order-service-v2
  uri: lb://order-service-v2
  predicates:
    - Path=/api/orders/**
    - Header=API-Version, v2    # only route if header present

- id: order-service-v1
  uri: lb://order-service-v1
  predicates:
    - Path=/api/orders/**       # fallback — no version header
```

### Method-Based Routing

```yaml
# Route reads and writes to different instances
- id: products-read
  uri: lb://product-service-read     # read replicas
  predicates:
    - Path=/api/v1/products/**
    - Method=GET,HEAD

- id: products-write
  uri: lb://product-service-write    # write instances
  predicates:
    - Path=/api/v1/products/**
    - Method=POST,PUT,PATCH,DELETE
```

---

## Internals — Rate Limiting

Rate limiting at the gateway protects all downstream services from being overwhelmed by a single client.

### Algorithms

**Token Bucket (most common):**
```
Each client has a bucket with capacity N tokens
Tokens refill at rate R per second
Each request consumes 1 token
If bucket empty — request rejected with 429

Example:
  Capacity: 100 tokens
  Refill:   10 tokens/second
  
  Client fires 100 requests instantly:
    → All 100 succeed (bucket empties)
    → 101st request → 429 Too Many Requests
    → After 1 second → 10 new tokens → 10 more requests allowed
    
  Allows bursts up to bucket capacity
  Sustained rate limited to refill rate
```

**Sliding Window:**
```
Count requests in the last N seconds for this client
If count > limit → reject

Rolling window — more accurate than fixed window
No burst allowed beyond the limit at any moment

Example:
  Limit: 100 requests per 60 seconds
  Client made 99 requests in last 59 seconds
  One more request — allowed (100 total in window)
  Next request — rejected (101 in window)
  As old requests age out — capacity restored gradually
```

**Fixed Window:**
```
Count requests in the current time window (e.g. current minute)
Reset count at window boundary

Problem — window boundary spike:
  Limit: 100/minute
  Client sends 100 at 11:59:55 (allowed)
  Client sends 100 at 12:00:05 (allowed — new window)
  200 requests in 10 seconds — burst through the limit
  
Simple to implement but has the boundary problem
```

### Rate Limiting Implementation in Spring Cloud Gateway

```java
@Configuration
public class RateLimitConfig {

    // Rate limit by userId from JWT (authenticated users)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest()
                .getHeaders()
                .getFirst("X-User-Id");  // set by auth filter upstream
            return Mono.just(userId != null ? userId : "anonymous");
        };
    }

    // Rate limit by IP (unauthenticated requests)
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest()
                .getRemoteAddress()
                .getAddress()
                .getHostAddress()
        );
    }
}
```

```yaml
# Apply rate limiting per route
routes:
  - id: order-service
    uri: lb://order-service
    predicates:
      - Path=/api/v1/orders/**
    filters:
      - name: RequestRateLimiter
        args:
          redis-rate-limiter.replenishRate: 10     # tokens/second
          redis-rate-limiter.burstCapacity: 50     # max burst
          redis-rate-limiter.requestedTokens: 1    # cost per request
          key-resolver: "#{@userKeyResolver}"
```

**Distributed rate limiting — why Redis:**
```
Problem:
  Gateway runs 3 instances for HA
  Client makes 100 requests, routed across all 3 instances
  Each instance thinks client made ~33 requests
  → Rate limit never triggers

Solution — Redis as shared counter:
  All gateway instances share one Redis counter per client
  Atomic increment + check = correct distributed count
  Redis INCR is atomic — no race conditions
  
Script (Lua — atomic execution):
  local count = redis.call('INCR', key)
  if count == 1 then
    redis.call('EXPIRE', key, window_seconds)
  end
  return count
```

---

## Internals — Circuit Breaking at the Gateway

The gateway is the perfect place for circuit breakers — it can detect that a downstream service is failing and stop routing traffic to it:

```
Normal state (CLOSED):
  All requests flow through to Order Service
  Gateway tracks: success rate, response times

Degraded state — failures accumulate:
  Order Service starts returning 500s
  Gateway records: 10 failures in 30 seconds

OPEN state — circuit trips:
  Gateway stops routing to Order Service
  Immediately returns fallback response to clients
  No traffic reaches the struggling service
  Service gets time to recover

HALF-OPEN state — probe recovery:
  After 30 seconds, gateway sends one test request
  If it succeeds → circuit CLOSES → traffic resumes
  If it fails → circuit re-OPENS → wait again
```

```yaml
# Circuit breaker configuration
filters:
  - name: CircuitBreaker
    args:
      name: orderServiceCircuitBreaker
      fallbackUri: forward:/fallback/orders
      
  - name: Retry
    args:
      retries: 3
      statuses: SERVICE_UNAVAILABLE, GATEWAY_TIMEOUT
      methods: GET          # only retry safe methods
      backoff:
        firstBackoff: 100ms
        maxBackoff: 1000ms
        factor: 2            # exponential: 100ms, 200ms, 400ms
```

```java
// Fallback controller — what clients see when circuit is open
@RestController
@RequestMapping("/fallback")
public class FallbackController {

    @GetMapping("/orders")
    public ResponseEntity<Map<String, Object>> ordersFallback() {
        return ResponseEntity.status(503).body(Map.of(
            "error", "ORDER_SERVICE_UNAVAILABLE",
            "message", "Order service is temporarily unavailable. Please try again shortly.",
            "retryAfter", 30
        ));
    }

    @GetMapping("/products")
    public ResponseEntity<?> productsFallback() {
        // Return cached product list from Redis if available
        List<Product> cached = productCacheService.getCachedProducts();
        if (!cached.isEmpty()) {
            return ResponseEntity.ok(Map.of(
                "data", cached,
                "fromCache", true,
                "warning", "Live data temporarily unavailable"
            ));
        }
        return ResponseEntity.status(503).body(Map.of(
            "error", "PRODUCT_SERVICE_UNAVAILABLE"
        ));
    }
}
```

---

## Internals — Authentication Filter

The gateway's auth filter runs on every request before routing:

```java
@Component
@Order(1)  // runs first
public class JwtAuthenticationFilter implements GlobalFilter {

    private final JwtTokenValidator jwtValidator;
    private final List<String> publicPaths = List.of(
        "/api/v1/auth/",
        "/api/v1/products/",   // public product browsing
        "/api/v1/search/",
        "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getPath().toString();

        // 1. Skip auth for public paths
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }

        // 2. Extract JWT from Authorization header
        String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorizedResponse(exchange, "Missing or malformed Authorization header");
        }

        String token = authHeader.substring(7);

        // 3. Validate JWT
        try {
            JwtClaims claims = jwtValidator.validate(token);

            // 4. Forward userId and roles as internal headers
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id",    claims.getUserId())
                .header("X-User-Email", claims.getEmail())
                .header("X-User-Roles", String.join(",", claims.getRoles()))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (TokenExpiredException e) {
            return unauthorizedResponse(exchange, "Token expired");
        } catch (InvalidTokenException e) {
            return unauthorizedResponse(exchange, "Invalid token");
        }
    }

    private Mono<Void> unauthorizedResponse(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        
        String body = """
            {"error":"UNAUTHORIZED","message":"%s"}
            """.formatted(message);
        
        DataBuffer buffer = response.bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));
        return response.writeWith(Mono.just(buffer));
    }

    private boolean isPublicPath(String path) {
        return publicPaths.stream().anyMatch(path::startsWith);
    }
}
```

---

## Gateway vs Load Balancer — Know the Difference

This is a common interview confusion:

```
Load Balancer (L4/L7):
  Distributes traffic across multiple instances of ONE service
  Works at TCP (L4) or HTTP (L7) level
  Minimal logic — round robin, least connections, health checks
  Does NOT understand application-level concerns
  e.g. AWS ALB, Nginx, HAProxy

API Gateway:
  Routes traffic to MANY different services
  Rich application-level logic — auth, rate limiting, transformation
  Understands HTTP semantics deeply
  Often sits BEHIND a load balancer
  e.g. Spring Cloud Gateway, Kong, AWS API Gateway

In practice:
  Internet → Load Balancer → API Gateway instances → Microservices
                                    ↑
                    Multiple gateway instances for HA
                    Load balancer distributes across them
```

---

## Gateway Patterns — Advanced

### BFF Gateway — Backend for Frontend

```
Mobile App  ──► Mobile BFF Gateway ──► internal services
Web App     ──► Web BFF Gateway    ──► internal services
3rd Party   ──► Public API Gateway ──► internal services

Each gateway is tailored to its client:
  Mobile BFF:
    Aggregates multiple service calls into one response
    Returns minimal fields (mobile screen needs less data)
    Handles mobile-specific auth (refresh token via push)

  Web BFF:
    Returns richer responses
    Handles session-based auth
    Serves server-side rendered pages

  Public API:
    Strict rate limiting
    API key auth
    Versioned endpoints
    Developer-friendly error messages
```

### Gateway Aggregation — Reducing Round Trips

```java
// Gateway aggregates multiple service calls into one response
// Client makes ONE request — gateway fans out to multiple services

@GetMapping("/api/v1/dashboard")
public Mono<DashboardResponse> getDashboard(
        @RequestHeader("X-User-Id") String userId) {

    // Fan out — all three calls happen in parallel
    Mono<List<Order>> recentOrders = orderServiceClient
        .getRecentOrders(userId, 5);

    Mono<UserProfile> userProfile = userServiceClient
        .getProfile(userId);

    Mono<List<Recommendation>> recommendations = recommendationServiceClient
        .getRecommendations(userId, 10);

    // Zip — combine when all three complete
    return Mono.zip(recentOrders, userProfile, recommendations)
        .map(tuple -> DashboardResponse.builder()
            .orders(tuple.getT1())
            .profile(tuple.getT2())
            .recommendations(tuple.getT3())
            .build());
}
```

Client makes one HTTP request instead of three — reduces mobile battery usage and round-trip latency.

### Canary Routing at the Gateway

```yaml
# Route 5% of traffic to new version for canary testing
routes:
  - id: order-service-canary
    uri: lb://order-service-v2        # new version
    predicates:
      - Path=/api/v1/orders/**
      - Weight=order-group, 5         # 5% of traffic

  - id: order-service-stable
    uri: lb://order-service-v1        # stable version
    predicates:
      - Path=/api/v1/orders/**
      - Weight=order-group, 95        # 95% of traffic
```

Zero infrastructure changes needed — canary deployment managed entirely through gateway routing rules.

---

## Observability at the Gateway

The gateway is the perfect observability collection point — every request passes through it:

```java
@Component
public class RequestLoggingFilter implements GlobalFilter {

    private final MeterRegistry meterRegistry;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();
        ServerHttpRequest request = exchange.getRequest();

        String requestId = UUID.randomUUID().toString();
        String path = request.getPath().toString();
        String method = request.getMethod().name();
        String userId = request.getHeaders().getFirst("X-User-Id");

        // Propagate request ID for distributed tracing
        ServerHttpRequest mutated = request.mutate()
            .header("X-Request-Id", requestId)
            .header("X-Trace-Id",   requestId)
            .build();

        return chain.filter(exchange.mutate().request(mutated).build())
            .doFinally(signal -> {
                long duration = System.currentTimeMillis() - startTime;
                int statusCode = exchange.getResponse()
                    .getStatusCode().value();

                // Log every request
                log.info("method={} path={} status={} duration={}ms userId={} requestId={}",
                    method, path, statusCode, duration, userId, requestId);

                // Emit metrics
                meterRegistry.timer("gateway.request.duration",
                    "path",   path,
                    "method", method,
                    "status", String.valueOf(statusCode)
                ).record(Duration.ofMillis(duration));

                // Alert on errors
                if (statusCode >= 500) {
                    meterRegistry.counter("gateway.errors",
                        "path", path, "status", String.valueOf(statusCode)
                    ).increment();
                }
            });
    }
}
```

---

## Real-World Example — ShopSphere Complete Gateway Architecture

```
                    ┌──────────────────────────────────────────┐
                    │         ShopSphere API Gateway            │
                    │        (Spring Cloud Gateway)             │
                    │                                           │
                    │  Filters (in order):                      │
                    │    1. Request Logging Filter               │
                    │    2. Rate Limiting Filter (Redis)         │
                    │    3. JWT Authentication Filter            │
                    │    4. CORS Filter                         │
                    │    5. Request ID Propagation Filter        │
                    │                                           │
                    │  Routes:                                  │
                    │    /api/v1/auth/**    → auth-service      │
                    │    /api/v1/users/**   → user-service      │
                    │    /api/v1/products/**→ product-service   │
                    │    /api/v1/orders/**  → order-service     │
                    │    /api/v1/search/**  → search-service    │
                    │    /api/v1/cart/**    → cart-service      │
                    │    /api/v1/payments/**→ payment-service   │
                    │    /api/v1/admin/**   → admin-service     │
                    │    /api/v1/reviews/** → review-service    │
                    │    /ws/**             → notification-svc  │
                    └──────────────────────┬───────────────────┘
                                           │
                    ┌──────────────────────┼───────────────────┐
                    │                      │                    │
              order-service          product-service      user-service
              port: 8081             port: 8082           port: 8083
              replicas: 3            replicas: 2          replicas: 2
```

---

## Interview Q&A

**Q: What is an API Gateway and why is it necessary in microservices?**
An API Gateway is the single entry point for all client traffic in a microservices system. It centralises cross-cutting concerns — authentication, rate limiting, SSL termination, routing, logging, circuit breaking — that every service would otherwise have to implement independently. Without it, clients must know the address of every service, every service duplicates infrastructure logic, and there is no unified place to enforce security policies. With it, services focus purely on business logic and the gateway enforces consistency across all entry points.

**Q: What is the difference between an API Gateway and a load balancer?**
A load balancer distributes traffic across multiple instances of a single service — it is simple, fast, and operates at the transport or HTTP level with minimal logic. An API Gateway routes traffic to many different services based on rich application-level rules — path, method, header — and executes complex logic like authentication, rate limiting, and request transformation. In production they complement each other: a load balancer distributes traffic across multiple API Gateway instances for high availability, and the gateway handles all application-level routing and policy enforcement downstream.

**Q: How does rate limiting work at the API Gateway and how do you make it work across multiple gateway instances?**
Rate limiting at the gateway tracks request counts per client identity — typically userId or IP — and rejects requests exceeding the configured threshold with a 429 response. The token bucket algorithm allows bursts up to a capacity limit with a sustained refill rate. Making it work across multiple gateway instances requires a shared atomic counter — Redis is the standard solution. All gateway instances increment and check the same Redis counter per client using a Lua script for atomicity. Without Redis, each instance tracks its own counter and the effective rate limit becomes N times the configured limit for N instances.

**Q: What happens at the gateway when a downstream service is down?**
The circuit breaker pattern handles this. The gateway tracks failure rates for each downstream service. When failures exceed a threshold — for example 10 failures in 30 seconds — the circuit opens and the gateway stops routing traffic to that service entirely, immediately returning a fallback response to clients. This gives the failing service time to recover without being hammered by traffic. After a configured wait period the circuit moves to half-open, allowing one probe request. If it succeeds the circuit closes and normal routing resumes. Fallback responses can be static error messages, cached data from Redis, or degraded partial responses.

**Q: How would you implement canary deployments using an API Gateway?**
Spring Cloud Gateway supports weighted routing — you configure two routes for the same path pointing to the old and new service versions with weights like 95 and 5. The gateway distributes 5% of traffic to the new version while 95% continues to the stable version. You monitor error rates, latency, and business metrics for the canary. If metrics are healthy, you gradually shift weight — 10%, 25%, 50%, 100%. If metrics degrade you immediately set the canary weight to 0 with no deployment rollback needed. This is pure traffic management at the gateway layer.

---

