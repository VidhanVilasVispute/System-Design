# Stage 2 — APIs & Communication Patterns

## Topic 1: What is an API

## Theory

**API** stands for **Application Programming Interface**. At its core, an API is a **contract** — a defined agreement between two pieces of software on how they will communicate.

The word "interface" is the key. Just like a power socket is an interface — you don't need to know how electricity is generated to plug something in — an API hides the internal implementation and exposes only what the consumer needs to interact with.

**Three levels at which APIs exist:**
```
1. Library/Framework APIs
   e.g. Java's Collections API — List.add(), Map.get()
   You call code within the same process

2. Operating System APIs
   e.g. POSIX — open(), read(), write(), fork()
   Your program talks to the OS kernel

3. Web/Network APIs  ← what we mean in system design
   e.g. Stripe API, Twitter API, your own microservice APIs
   Services talk to each other over a network
```

In system design, when we say API we almost always mean **Web APIs** — services exposing functionality over HTTP.

---

## Internals — What an API Actually Is

An API is three things simultaneously:

**1. A contract — what you promise to support:**
```
Endpoint:  POST /orders
Input:     { userId, items[], paymentMethodId }
Output:    { orderId, status, total }
Errors:    400 if items empty, 402 if payment fails,
           404 if userId not found
```
This contract is what your consumers depend on. Breaking it without versioning breaks them.

**2. An abstraction — hiding implementation:**
```
Consumer calls:  POST /orders
                       ↓
           (consumer has no idea about any of this)
                       ↓
  Order Service validates input
       ↓
  Checks inventory via Inventory Service
       ↓
  Charges card via Stripe
       ↓
  Writes to PostgreSQL
       ↓
  Publishes OrderCreated event to Kafka
       ↓
  Returns { orderId: "o-789", status: "CONFIRMED" }
```
The consumer gets a clean result. The complexity is invisible.

**3. A boundary — separating concerns:**
- The API is the wall between your service and everything else
- Internal data structures, DB schema, business logic — all stay behind the wall
- Only what is explicitly exposed in the API is accessible

---

## API Design Principles — What Makes a Good API

This is heavily asked in interviews. A good API has these properties:

**1. Intuitive — self-documenting:**
```
Bad:   POST /doThing     ← what thing?
Bad:   GET  /getData     ← what data?
Good:  POST /orders      ← clear resource
Good:  GET  /orders/{id} ← clear intent
```

**2. Consistent:**
```
Bad — mixed conventions:
  GET  /getUser/{id}
  POST /create-order
  PUT  /products/update/{id}

Good — uniform convention:
  GET    /users/{id}
  POST   /orders
  PUT    /products/{id}
```

**3. Versioned — never break consumers:**
```
/api/v1/orders   ← existing consumers stay here
/api/v2/orders   ← new breaking change goes here

Both run simultaneously during migration window
```

**4. Idempotent where possible:**
```
POST /orders                     ← not idempotent, calling twice creates two orders
POST /orders + Idempotency-Key: abc-123   ← idempotent, same key = same result

Client can safely retry on network failure without fear of duplicate orders
```

Idempotency keys are critical for payment and order APIs. Stripe, PayPal, and every serious payment processor requires them.

**5. Paginated — never return unbounded lists:**
```
Bad:  GET /products    → returns all 500,000 products
Good: GET /products?page=1&size=20&sort=price:asc

Response includes:
{
  "data": [...20 products],
  "pagination": {
    "page": 1,
    "size": 20,
    "total": 500000,
    "hasNext": true
  }
}
```

**6. Proper error responses — tell the client what went wrong:**
```json
Bad:
HTTP 400
{ "error": "bad request" }

Good:
HTTP 422
{
  "error": "VALIDATION_FAILED",
  "message": "Order validation failed",
  "details": [
    { "field": "items", "issue": "Cart cannot be empty" },
    { "field": "paymentMethodId", "issue": "Payment method not found" }
  ],
  "requestId": "req-abc-123"
}
```
Always include a `requestId` — it links the error to your logs and makes debugging across services possible.

**7. Secure by default:**
- Every endpoint authenticated unless explicitly public
- Authorisation checked at resource level — not just "is the user logged in" but "does this user own this order"
- Rate limited to prevent abuse
- Input validated and sanitised before processing

---

## API Gateway — The Front Door

In a microservices architecture, clients do not call individual services directly. Everything goes through an **API Gateway**:

```
Mobile App  ─┐
Web Frontend ─┼──► API Gateway ──► Order Service
Third-party  ─┘         │
                         ├──► Product Service
                         ├──► User Service
                         └──► Payment Service
```

The API Gateway handles **cross-cutting concerns** — things every service needs but should not implement itself:

```
Incoming request
       ↓
  Rate Limiting      ← reject if client exceeds quota
       ↓
  Authentication     ← validate JWT, reject if invalid
       ↓
  Authorisation      ← check permissions
       ↓
  Request Logging    ← log request ID, path, client
       ↓
  SSL Termination    ← decrypt HTTPS, forward HTTP internally
       ↓
  Routing            ← forward to correct microservice
       ↓
  Response Logging   ← log status code, latency
       ↓
  Response to client
```

Without an API Gateway, every service would need to duplicate all of this logic. With it, services focus purely on business logic.

**ShopSphere's API Gateway** is Spring Cloud Gateway — it handles routing, JWT validation, and rate limiting for all 22 services behind a single entry point at port 8080.

---

## API Documentation — OpenAPI / Swagger

A contract is useless if it is not documented. The industry standard is **OpenAPI Specification (formerly Swagger)**:

```yaml
openapi: 3.0.0
paths:
  /orders:
    post:
      summary: Create a new order
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          description: Invalid input
        '402':
          description: Payment failed
```

In Spring Boot this is generated automatically with `springdoc-openapi`:
```java
@Operation(summary = "Create a new order")
@ApiResponse(responseCode = "201", description = "Order created")
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) { ... }
```

Swagger UI at `/swagger-ui.html` gives you an interactive API explorer — every consumer can test your API without writing a line of code.

---

## Real-World Example — ShopSphere API Design

**Well-designed Order Service API:**

```
Public endpoints (no auth):
  GET  /api/v1/products              ← list products
  GET  /api/v1/products/{id}         ← product detail
  GET  /api/v1/search?q={query}      ← search

Authenticated endpoints:
  POST /api/v1/orders                ← place order
  GET  /api/v1/orders/{id}           ← get my order
  POST /api/v1/orders/{id}/cancel    ← cancel order
  GET  /api/v1/users/me              ← my profile
  PUT  /api/v1/users/me              ← update profile

Admin endpoints (admin role required):
  GET  /api/v1/admin/orders          ← all orders
  PUT  /api/v1/admin/orders/{id}     ← update any order
  POST /api/v1/admin/products        ← create product
```

**Idempotent order creation:**
```
POST /api/v1/orders
Idempotency-Key: client-generated-uuid-123
Authorization: Bearer <jwt>

{
  "items": [{"productId": "p-456", "quantity": 2}],
  "paymentMethodId": "pm-789"
}
```
If the client retries due to a network timeout, the server checks if `client-generated-uuid-123` was already processed — returns the original response instead of creating a duplicate order.

---

## Interview Q&A

**Q: What is an API and why does it matter in distributed systems?**
An API is a contract between two software components defining how they communicate — what inputs are accepted, what outputs are returned, and what errors can occur. In distributed systems it matters because it is the only stable interface between services. Internal implementations can change freely — DB schema, business logic, language — as long as the API contract is honoured. Without well-defined APIs, microservices become tightly coupled and changes cascade unpredictably.

**Q: What makes a good API design?**
A good API is intuitive with consistent resource naming and HTTP verb semantics, versioned so existing consumers never break, idempotent on write operations so retries are safe, paginated so responses are bounded, returns structured errors with enough context to debug, and is secured with authentication and authorisation at every endpoint. Documentation via OpenAPI is also non-negotiable for any API consumed by more than one team.

**Q: What is an idempotency key and when would you use it?**
An idempotency key is a unique client-generated identifier sent with a request, typically in a header. The server stores the result of the first execution against this key and returns the cached result for any subsequent request with the same key. You use it for non-idempotent operations like order placement or payment processing — where a network timeout might cause the client to retry and risk creating duplicates. Stripe requires idempotency keys on all payment API calls for exactly this reason.

**Q: What is an API Gateway and what does it do?**
An API Gateway is a single entry point for all client requests in a microservices architecture. It handles cross-cutting concerns that every service needs but should not implement individually — authentication, authorisation, rate limiting, SSL termination, request routing, and logging. Without it, every microservice duplicates this logic. With it, services focus purely on business logic and the gateway enforces consistency across all endpoints.

**Q: How would you version a REST API without breaking existing consumers?**
Introduce the new version under a new path prefix — `/api/v2/orders` — while keeping `/api/v1/orders` running unchanged. Run both versions simultaneously during a migration window, communicate the deprecation timeline clearly to consumers, and only retire v1 once all consumers have migrated. Never make breaking changes — field removals, type changes, renamed fields — in place on an existing version. Additive changes — new optional fields, new endpoints — can be made to existing versions without a new version.

---

Say **"next"** when ready for Topic 2 — REST API Design (deep dive).
