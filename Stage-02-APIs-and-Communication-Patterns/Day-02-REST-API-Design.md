# Stage 2 — Topic 2: REST API Design (Deep Dive)

## Theory

We touched on REST in the previous topic. Now we go deep — the internals, the patterns, the mistakes, and the nuances that separate a junior API design from a senior one.

REST was defined by Roy Fielding in his 2000 PhD dissertation. Most APIs that call themselves RESTful violate at least one of his constraints. That is fine in practice — but you need to know what the constraints are, why they exist, and when violating them is a deliberate trade-off vs accidental sloppiness.

**The six REST constraints:**
```
1. Client-Server       — separation of concerns, client and server evolve independently
2. Stateless           — no session state on the server, every request is self-contained
3. Cacheable           — responses must declare whether they are cacheable
4. Uniform Interface   — consistent resource identification and manipulation
5. Layered System      — client cannot tell if it is talking to the real server or a proxy
6. Code on Demand      — server can send executable code to client (optional, rarely used)
```

In practice, constraints 1, 2, 3, and 4 are the ones that matter day to day.

---

## Internals — Statelessness in Depth

**What stateless actually means:**

Every request must contain **all information** the server needs to process it. The server stores nothing about the client between requests.

```
Stateful (bad):
  Request 1: POST /login        → server stores session in memory: { userId: "u-123" }
  Request 2: GET  /orders       → server looks up session to know who is asking
  Request 3: POST /orders       → server uses session for userId

  Problem: Request 2 and 3 MUST hit the same server instance
           Horizontal scaling is broken
           Server crashes → all sessions lost

Stateless (correct):
  Request 1: POST /login        → server returns JWT token to client
  Request 2: GET  /orders       → client sends JWT in header, server reads userId from token
  Request 3: POST /orders       → client sends JWT in header, server reads userId from token

  Any server instance can handle any request
  Server crash loses nothing — client still has JWT
  Horizontal scaling works perfectly
```

JWT makes REST stateless — all identity information is encoded in the token itself, cryptographically signed so the server trusts it without storing anything.

---

## Resource Design — The Core Skill

The hardest part of REST design is **modelling your domain as resources**. Resources are nouns, not verbs.

**Rule 1 — URLs are nouns, HTTP methods are verbs:**
```
Bad  — verbs in URLs:
  POST /createOrder
  GET  /getProduct/123
  POST /deleteUser/456
  POST /updateOrderStatus

Good — nouns in URLs, verbs via HTTP methods:
  POST   /orders
  GET    /products/123
  DELETE /users/456
  PATCH  /orders/{id}/status
```

**Rule 2 — Plural nouns for collections:**
```
/orders      ← collection of all orders
/orders/123  ← single order with id 123
/products    ← collection
/products/456 ← single product
```

**Rule 3 — Hierarchical relationships via nesting (but max 2 levels deep):**
```
/orders/{orderId}/items              ← items belonging to an order
/orders/{orderId}/items/{itemId}     ← specific item in an order
/users/{userId}/addresses            ← addresses belonging to a user

Too deep — don't do this:
/users/{userId}/orders/{orderId}/items/{itemId}/reviews/{reviewId}
```

When nesting gets too deep, flatten it:
```
Instead of: /users/{userId}/orders/{orderId}/items/{itemId}
Use:        /order-items/{itemId}  (with userId and orderId as query params or in body)
```

**Rule 4 — Actions that don't fit CRUD go as sub-resources with POST:**
```
Cancel an order:      POST /orders/{id}/cancel
Approve a review:     POST /reviews/{id}/approve
Resend a notification: POST /notifications/{id}/resend
Verify an email:      POST /users/{id}/email/verify
Archive a product:    POST /products/{id}/archive
```

These are **state transitions** — not full resource replacements. Using POST with a verb sub-resource is the accepted REST pattern for actions.

---

## Query Parameters — When and How

Query parameters are for **filtering, sorting, searching, and pagination** — not for identifying a resource.

```
Filtering:
  GET /orders?status=PENDING
  GET /orders?status=CONFIRMED&userId=u-123
  GET /products?category=shoes&minPrice=50&maxPrice=200

Sorting:
  GET /products?sort=price:asc
  GET /orders?sort=createdAt:desc

Pagination:
  GET /products?page=2&size=20
  GET /orders?cursor=eyJpZCI6MTAwfQ==&size=20   ← cursor-based pagination

Search:
  GET /products?q=nike+running+shoes

Combining:
  GET /products?category=shoes&minPrice=50&sort=price:asc&page=1&size=20
```

**Offset pagination vs cursor pagination:**

```
Offset pagination:
  GET /products?page=2&size=20
  SQL: SELECT * FROM products LIMIT 20 OFFSET 20

  Simple to implement
  Problem: if a new product is inserted while user is browsing,
           page 2 might show a duplicate or skip a product
  Problem: OFFSET 10000 on a large table is slow — DB scans and discards 10,000 rows

Cursor pagination:
  GET /products?cursor=eyJpZCI6MTAwfQ==&size=20
  Cursor encodes: { "id": 100 } — the last seen item
  SQL: SELECT * FROM products WHERE id > 100 LIMIT 20

  Stable — insertions don't cause duplicates or skips
  Fast — indexed seek instead of offset scan
  No random access — you cannot jump to page 50
  Best for: infinite scroll feeds, real-time data
```

For ShopSphere's product listing — offset pagination is fine (data doesn't change rapidly). For the order feed or notification feed — cursor pagination is better.

---

## Request & Response Design

**Request body — be explicit about required vs optional:**
```json
POST /orders
{
  "items": [                          ← required, array
    {
      "productId": "p-456",           ← required
      "quantity": 2,                  ← required
      "note": "Gift wrap please"      ← optional
    }
  ],
  "paymentMethodId": "pm-789",        ← required
  "shippingAddressId": "addr-321",    ← required
  "promoCode": "SAVE10"               ← optional
}
```

**Response body — consistent envelope:**
```json
Success response:
{
  "data": { ... },           ← actual payload always under "data"
  "requestId": "req-abc",    ← for debugging and log correlation
  "timestamp": "2026-04-08T10:30:00Z"
}

List response:
{
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "size": 20,
    "total": 500,
    "hasNext": true,
    "hasPrev": false
  },
  "requestId": "req-abc"
}

Error response:
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "details": [
      { "field": "items", "issue": "Cannot be empty" }
    ]
  },
  "requestId": "req-abc",
  "timestamp": "2026-04-08T10:30:00Z"
}
```

Consistent envelope across all endpoints means clients write generic error handling once — they always know where to find the data, pagination, and errors.

---

## HATEOAS — Hypermedia as the Engine of Application State

HATEOAS is the most misunderstood and most ignored REST constraint. Roy Fielding considers it required for true REST. Most APIs ignore it.

**What it means:** Responses include links to related actions — the client discovers what it can do next from the response itself, not from documentation.

```json
GET /orders/o-789
{
  "orderId": "o-789",
  "status": "CONFIRMED",
  "total": 1299.99,
  "_links": {
    "self":   { "href": "/orders/o-789" },
    "cancel": { "href": "/orders/o-789/cancel", "method": "POST" },
    "track":  { "href": "/shipments/s-456", "method": "GET" },
    "invoice":{ "href": "/orders/o-789/invoice", "method": "GET" }
  }
}
```

The client does not need to know that cancellation is at `/orders/{id}/cancel` — the response tells it. If the order is already shipped, the `cancel` link simply does not appear — the client knows the action is unavailable without hardcoding business rules.

**In practice:** most teams find HATEOAS complex to implement and maintain. It is more common in mature public APIs (GitHub API, PayPal API) than internal microservices. Know what it is — you will not always implement it, but you should be able to discuss it.

---

## Common REST Design Mistakes

**Mistake 1 — Using GET for state-changing operations:**
```
Bad:  GET /orders/cancel?id=123    ← GET must be safe and idempotent
Good: POST /orders/123/cancel
```
GET requests are cached by browsers, CDNs, proxies. A state-changing GET is a disaster waiting to happen.

**Mistake 2 — Returning 200 for errors:**
```
Bad:
HTTP 200 OK
{ "success": false, "error": "User not found" }

Good:
HTTP 404 Not Found
{ "error": { "code": "USER_NOT_FOUND", "message": "User u-999 not found" } }
```
Status codes exist for a reason — use them. Monitoring systems, load balancers, and clients rely on them.

**Mistake 3 — Exposing internal DB IDs:**
```
Bad:  /orders/1, /orders/2, /orders/3
      ← exposes sequential DB primary key
      ← competitor can enumerate all orders
      ← reveals order volume

Good: /orders/ord_a3f9b2c1
      ← opaque UUID or prefixed ID
      ← no information leakage
```

**Mistake 4 — No rate limiting on write endpoints:**
```
POST /orders with no rate limit
→ competitor scripts 10,000 fake orders per second
→ inventory depleted, system overwhelmed
```

**Mistake 5 — Inconsistent date formats:**
```
Bad:  "createdAt": "04/08/2026"         ← ambiguous, US vs EU format
Bad:  "createdAt": 1712566200           ← unix timestamp, no timezone
Good: "createdAt": "2026-04-08T10:30:00Z"  ← ISO 8601, always UTC
```

---

## Real-World Example — ShopSphere Order Service API

**Full REST contract for Order Service:**

```
Base URL: /api/v1

─── Orders ───────────────────────────────────────────────

POST   /orders
  Auth:     Required
  Body:     { items, paymentMethodId, shippingAddressId, promoCode? }
  Headers:  Idempotency-Key: <client-uuid>
  Returns:  201 { orderId, status, total, estimatedDelivery }
  Errors:   400 validation, 402 payment failed, 409 item out of stock

GET    /orders
  Auth:     Required
  Params:   status?, sort?, cursor?, size?
  Returns:  200 { data: [orders], pagination }

GET    /orders/{orderId}
  Auth:     Required (must own the order or be admin)
  Returns:  200 { order details + _links }
  Errors:   404 not found, 403 not your order

POST   /orders/{orderId}/cancel
  Auth:     Required (must own the order)
  Returns:  200 { orderId, status: "CANCELLED" }
  Errors:   409 already shipped, cannot cancel

GET    /orders/{orderId}/items
  Auth:     Required
  Returns:  200 { data: [items] }

─── Admin Orders ──────────────────────────────────────────

GET    /admin/orders
  Auth:     Required + ADMIN role
  Params:   status?, userId?, dateFrom?, dateTo?, page?, size?
  Returns:  200 { data: [orders], pagination }

PATCH  /admin/orders/{orderId}/status
  Auth:     Required + ADMIN role
  Body:     { status: "PROCESSING" | "SHIPPED" | "DELIVERED" }
  Returns:  200 { orderId, status }
```

---

## Interview Q&A

**Q: What is the difference between PUT and PATCH?**
PUT replaces the entire resource — you send the complete new state and the server overwrites everything. If you omit a field, it gets set to null or removed. PATCH partially updates a resource — you send only the fields you want to change and everything else stays the same. Use PUT when replacing a full document, PATCH when making targeted updates like changing just an order status or a user's email.

**Q: How would you design pagination for a high-traffic feed in ShopSphere?**
Use cursor-based pagination instead of offset. Offset pagination becomes slow on large tables because the database must scan and discard thousands of rows to reach the offset. It also produces inconsistent results when new items are inserted — users see duplicates or skip items. Cursor pagination uses the last seen item's ID as a bookmark, allowing the DB to seek directly to that position via an index. The trade-off is no random page access, which is acceptable for feed-style UIs using infinite scroll.

**Q: What is HATEOAS and do you need it?**
HATEOAS means responses include hypermedia links describing what actions are available next — cancelling an order, tracking a shipment, downloading an invoice. The client discovers the API dynamically from responses rather than hardcoding URLs. Roy Fielding considers it required for true REST. In practice most teams skip it for internal APIs because of implementation complexity, but it adds real value for public APIs consumed by many external clients — it makes the API self-documenting and allows server-side URL structure to change without breaking clients.

**Q: Why should you never use sequential integer IDs in public API URLs?**
Sequential IDs leak information — a competitor can enumerate all your orders to estimate your transaction volume. They also enable enumeration attacks where an attacker iterates through IDs to access resources they should not. Use opaque identifiers — UUIDs, or prefixed IDs like `ord_a3f9b2c1` — which reveal nothing about internal structure or volume.

**Q: How do you handle breaking vs non-breaking API changes?**
Non-breaking changes — adding optional fields, adding new endpoints, adding new optional query parameters — can be deployed to existing versions without a new version. Breaking changes — removing fields, renaming fields, changing types, altering required fields — require a new version. Introduce it under a new path prefix like `/api/v2`, run both versions simultaneously during a migration window, communicate deprecation dates, and retire the old version only after all consumers have migrated. Never silently break an existing version.

---

Say **"next"** when ready for Topic 3 — gRPC (deep dive).
