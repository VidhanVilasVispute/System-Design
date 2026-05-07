## Topic 4 — API Composition

### The Problem First

In a monolith, fetching data for a single page is easy:

```
"Load Order Details Page" → one method call →
  joins User table + Order table + Product table + Payment table
  → returns everything in one response
```

In microservices, that data is **spread across multiple services**, each owning its own DB:

```
Order Details Page needs:
  - Order info        → Order Service
  - Customer name     → User Service
  - Product details   → Product Service
  - Payment status    → Payment Service

No single service has all of it.
No shared DB to JOIN across.
```

How does the client get all this in one response? That's the **API Composition** problem.

---

### What is API Composition?

A **composer** (API Gateway or a dedicated aggregator service) calls multiple downstream services, collects their responses, merges them, and returns a single unified response to the client.

```
Client
  │
  │  GET /orders/123/details
  ▼
┌─────────────────────────────┐
│     API Composer            │
│  (Gateway or BFF service)   │
└──┬──────┬──────┬────────────┘
   │      │      │
   ▼      ▼      ▼
Order   User   Product
Service Service Service
   │      │      │
   └──────┴──────┘
          │
          ▼
   Merged Response
   returned to client
```

The client makes **one call**. The composer makes **multiple internal calls** and assembles the result.

---

### Two Composition Strategies

#### Strategy 1 — Sequential Composition

Each call depends on the result of the previous one.

```
Step 1: Call Order Service → get orderId, userId, [productId1, productId2]
                                    │
Step 2: Call User Service with userId → get customerName, email
                                    │
Step 3: Call Product Service with productId1, productId2 → get product details
                                    │
Step 4: Merge all → return to client

Total latency = T(order) + T(user) + T(product)
              = 50ms + 40ms + 60ms = 150ms
```

Use when: Call B depends on data returned by Call A.

---

#### Strategy 2 — Parallel Composition

Calls that don't depend on each other fire simultaneously.

```
                    ┌──── Order Service  (50ms)
Composer fires ─────┼──── User Service   (40ms)  → all at once
simultaneously      └──── Product Service(60ms)

Total latency = max(50, 40, 60) = 60ms  ← only the slowest one matters

vs Sequential = 150ms

3x faster for the same data.
```

Use when: Calls are independent of each other.

In Java this maps directly to `CompletableFuture`:

```java
CompletableFuture<OrderDTO> orderFuture = 
    CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));

CompletableFuture<UserDTO> userFuture = 
    CompletableFuture.supplyAsync(() -> userClient.getUser(userId));

CompletableFuture<List<ProductDTO>> productFuture = 
    CompletableFuture.supplyAsync(() -> productClient.getProducts(productIds));

// Wait for all three simultaneously
CompletableFuture.allOf(orderFuture, userFuture, productFuture).join();

OrderDetailsResponse response = merge(
    orderFuture.get(), 
    userFuture.get(), 
    productFuture.get()
);
```

---

### Where Does the Composer Live?

#### Option 1 — API Gateway as Composer

The gateway itself (Spring Cloud Gateway, Kong, Nginx) does the aggregation.

```
Client → API Gateway → calls Order + User + Product → merges → returns

Pro: No extra service to deploy
Con: Gateway becomes fat, logic-heavy, hard to test
     Gateways should route, not business-aggregate
```

#### Option 2 — Backend For Frontend (BFF)

A dedicated aggregator service per client type. The most production-grade approach.

```
Mobile App  → Mobile BFF  → calls relevant services → returns mobile-optimized payload
Web App     → Web BFF     → calls relevant services → returns web-optimized payload
3rd Party   → Public API  → calls relevant services → returns API-contract payload

Each BFF tailors the response shape to its consumer's exact needs.
Mobile gets a lighter payload (no fields it doesn't render).
Web gets richer data.
```

```
                    ┌──────────────┐
Mobile App ────────▶│  Mobile BFF  │──┐
                    └──────────────┘  │
                                      ├──▶ Order Service
                    ┌──────────────┐  ├──▶ User Service
Web App    ────────▶│   Web BFF    │──┤──▶ Product Service
                    └──────────────┘  │
                                      └──▶ Payment Service
```

---

### Handling Partial Failures

What if one downstream service is down during composition?

```
Composer calls Order + User + Product.
Product Service is down → 503.

Option 1: Fail the entire response → client gets error
          Simple but harsh. Order + User data was fine.

Option 2: Return partial response with nulls/defaults
          {
            "orderId": 123,
            "customer": { "name": "Vidhan" },
            "products": null,          ← Product Service was down
            "productsAvailable": false ← tell client explicitly
          }
          Client renders what it has, shows "product details unavailable"

Option 3: Return stale cached data for Product Service
          Serve last known product info from Redis cache
          Client gets a full response, possibly slightly stale
```

Option 2 + 3 combined is the production approach — **graceful degradation**.

---

### The N+1 Problem in Composition

A classic trap when composing in a loop:

```java
// WRONG — N+1 calls
List<Order> orders = orderService.getOrdersByUser(userId); // 1 call
for (Order order : orders) {
    Product p = productService.getProduct(order.getProductId()); // N calls!
}
// 1 order call + N product calls = N+1 total calls
// For 100 orders → 101 HTTP calls → slow, wasteful
```

Fix — **batch call**:

```java
// RIGHT — 2 calls total
List<Order> orders = orderService.getOrdersByUser(userId);          // 1 call
List<String> productIds = orders.stream()
    .map(Order::getProductId).collect(toList());
List<Product> products = productService.getProductsBatch(productIds); // 1 call
// Map products by ID and merge locally
```

Always design downstream services to support **batch endpoints** for composition use cases.

---

### API Composition vs GraphQL

GraphQL is essentially a standardized API Composition layer:

```
REST Composition (manual):
  Client calls BFF → BFF manually calls 4 services → BFF merges

GraphQL:
  Client sends one query describing exactly what fields it wants
  GraphQL server resolves each field via "resolvers" 
  (each resolver calls the relevant microservice)
  GraphQL merges and returns exactly what client asked for

GraphQL resolver map:
  Query.order       → calls Order Service
  Order.customer    → calls User Service with order.userId
  Order.products    → calls Product Service with order.productIds
  Order.payment     → calls Payment Service with order.paymentId
```

GraphQL is popular for BFF use cases because clients drive exactly what data they need — no over-fetching, no under-fetching.

---

### ShopSphere Lens

```
ShopSphere Order Details page needs:
  - Order data       → order-service
  - User profile     → user-service  
  - Product details  → product-service
  - Review summary   → review-service

Current approach: each service called separately from frontend (chatty)

Production approach: Add an order-aggregator BFF service

@GetMapping("/orders/{orderId}/details")
public OrderDetailsResponse getDetails(@PathVariable String orderId) {
    
    // Fire all calls in parallel
    CompletableFuture<OrderDTO> order = 
        CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));
    
    CompletableFuture<UserDTO> user = order.thenApplyAsync(o -> 
        userClient.getUser(o.getUserId()));   // depends on orderId result
    
    CompletableFuture<List<ProductDTO>> products = order.thenApplyAsync(o ->
        productClient.getProductsBatch(o.getProductIds())); // batch call
    
    CompletableFuture.allOf(user, products).join();
    
    return merge(order.get(), user.get(), products.get());
}
```

---

### Interview Questions

**Q1. What is API Composition and why is it needed in microservices?**

> In microservices, data is split across services with separate DBs — no shared JOIN is possible. API Composition aggregates responses from multiple services into a single client response, typically via an API Gateway or a dedicated BFF service.

**Q2. Sequential vs Parallel composition — when do you use each?**

> Sequential when call B needs data from call A's response. Parallel (using CompletableFuture.allOf) when calls are independent — total latency becomes the slowest call instead of the sum of all calls, dramatically improving response time.

**Q3. What is a BFF and why is it better than composing at the API Gateway?**

> BFF (Backend For Frontend) is a dedicated aggregator per client type. It keeps composition logic out of the gateway, allows different response shapes for mobile vs web, and is independently testable and deployable. The gateway should route and apply cross-cutting concerns — not contain business aggregation logic.

**Q4. How do you handle one downstream service being unavailable during composition?**

> Graceful degradation — return a partial response with nulls or cached stale data for the failed service, with an explicit flag indicating unavailability. This is better than failing the entire request when the missing data is non-critical. Use Resilience4j fallback methods to serve cached data when a service call fails.

---

Ready for **Topic 5 — Data Ownership** when you are.
