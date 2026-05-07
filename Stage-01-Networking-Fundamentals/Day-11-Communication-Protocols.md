# Topic 11: Communication Protocols (REST, gRPC, GraphQL)

## Theory

When two services need to talk to each other, they need to agree on three things:
- **What format** the data is in (JSON, XML, binary)
- **How to structure** requests and responses
- **What rules** govern the conversation

That agreement is the **communication protocol**. REST, gRPC, and GraphQL are the three dominant protocols in modern backend systems — each with a distinct philosophy, set of trade-offs, and ideal use case.

Choosing the wrong one is a real architectural mistake. Choosing the right one for the right layer of your system is a mark of an experienced engineer.

---

## REST — Representational State Transfer

### What it is

REST is not a protocol — it is an **architectural style**. It uses HTTP as the transport and defines a set of constraints for how resources should be exposed and manipulated.

**Core REST constraints:**
- **Stateless** — every request contains all information needed. Server holds no client session state.
- **Resource-based** — everything is a resource identified by a URL (`/orders/o-789`, `/users/u-123`)
- **HTTP verbs carry intent** — GET reads, POST creates, PUT replaces, PATCH updates, DELETE removes
- **Uniform interface** — consistent structure across all endpoints

### REST API Design — The Right Way

```
Resource:     /orders
Collection:   GET    /orders           → list all orders (paginated)
Single item:  GET    /orders/{id}      → get one order
Create:       POST   /orders           → create new order
Replace:      PUT    /orders/{id}      → replace entire order
Update:       PATCH  /orders/{id}      → partial update
Delete:       DELETE /orders/{id}      → remove order

Nested resource:
              GET    /orders/{id}/items         → items of a specific order
              POST   /orders/{id}/items         → add item to order

Actions that don't fit CRUD:
              POST   /orders/{id}/cancel        → cancel an order
              POST   /payments/{id}/refund      → trigger a refund
```

**REST response conventions:**
```json
GET /orders/o-789
→ 200 OK
{
  "orderId": "o-789",
  "status": "CONFIRMED",
  "total": 1299.99,
  "items": [...]
}

POST /orders
→ 201 Created
Location: /orders/o-790
{
  "orderId": "o-790",
  "status": "PENDING"
}

DELETE /orders/o-789
→ 204 No Content
```

### REST Problems at Scale

**Over-fetching:**
```
Client needs: product name + price only
REST returns: entire product object (name, price, description,
              images, tags, seller info, reviews, inventory...)
              
Wasted bandwidth and serialisation cost
```

**Under-fetching (N+1 problem):**
```
Client wants: order list with customer names

Step 1: GET /orders          → returns 20 orders, each with customerId
Step 2: GET /users/u-001     → get customer name
Step 3: GET /users/u-002     → get customer name
...
Step 21: GET /users/u-020    → get customer name

21 HTTP round trips to render one page
```

**Versioning complexity:**
```
/api/v1/orders    ← old clients still using this
/api/v2/orders    ← new structure
/api/v3/orders    ← breaking change again

Now you maintain 3 versions simultaneously
```

### When to use REST:
- Public APIs consumed by external developers
- Simple CRUD services with predictable access patterns
- When maximum compatibility and tooling ecosystem matter
- Mobile and web clients where HTTP caching is valuable

---

## gRPC — Google Remote Procedure Call

### What it is

gRPC is a **high-performance RPC framework** built by Google. Instead of thinking in resources and HTTP verbs, you think in **function calls** — you call a method on a remote service as if it were a local function.

**Built on:**
- **HTTP/2** — multiplexing, binary transport, header compression
- **Protocol Buffers (Protobuf)** — binary serialisation format, strongly typed, schema-first

### How gRPC Works

**Step 1 — Define your service in a `.proto` file:**
```protobuf
syntax = "proto3";

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc GetOrder    (GetOrderRequest)    returns (OrderResponse);
  rpc StreamOrderUpdates (GetOrderRequest) returns (stream OrderUpdate);
}

message CreateOrderRequest {
  string user_id   = 1;
  repeated OrderItem items = 2;
}

message OrderResponse {
  string order_id = 1;
  string status   = 2;
  double total    = 3;
}

message OrderUpdate {
  string order_id = 1;
  string status   = 2;
  string timestamp = 3;
}
```

**Step 2 — Generate client and server code:**
```bash
# Generates Java/Go/Python/etc stubs from .proto
protoc --java_out=. order.proto
```

The generated code handles all serialisation, deserialisation, and HTTP/2 transport. You just implement the business logic.

**Step 3 — Call it like a local function:**
```java
// gRPC client — looks like a local method call
OrderResponse response = orderStub.createOrder(
    CreateOrderRequest.newBuilder()
        .setUserId("u-123")
        .addItems(item)
        .build()
);
```

### Protobuf vs JSON

```
JSON:
{
  "orderId": "o-789",
  "status": "CONFIRMED",
  "total": 1299.99
}
Size: ~60 bytes, human-readable, loosely typed

Protobuf binary equivalent:
  0a 05 6f 2d 37 38 39 12 09 43 4f 4e 46 49 52 4d...
Size: ~20 bytes, binary, strongly typed, requires schema
```

- Protobuf is **3–10x smaller** than JSON
- Protobuf serialisation/deserialisation is **5–10x faster** than JSON
- But it is not human-readable — you need tooling to inspect messages

### gRPC Communication Patterns

This is one of gRPC's biggest advantages over REST — four communication modes:

```
1. Unary (like REST):
   Client sends one request → Server sends one response

2. Server Streaming:
   Client sends one request → Server streams multiple responses
   Use case: stream order status updates

3. Client Streaming:
   Client streams multiple messages → Server sends one response
   Use case: upload large file in chunks, bulk import

4. Bidirectional Streaming:
   Client and server both stream simultaneously
   Use case: real-time chat, live collaborative features
```

### gRPC Limitations:

- **Not browser-native** — browsers cannot call gRPC directly without gRPC-Web proxy
- **Schema coupling** — both sides must share the same `.proto` file — tight contract
- **Less human-friendly** — no curl, no Postman by default, harder to debug
- **Limited HTTP caching** — binary protocol does not benefit from standard HTTP cache infrastructure

### When to use gRPC:
- Internal service-to-service communication in microservices
- High-frequency, performance-critical paths (10,000+ req/s between services)
- When you need streaming (server push, bidirectional)
- Polyglot environments — same `.proto` generates clients in Java, Go, Python, etc.

---

## GraphQL

### What it is

GraphQL is a **query language for APIs** developed by Facebook. Instead of the server defining fixed endpoints that return fixed shapes, the **client specifies exactly what data it wants** in a single request.

One endpoint. Client-driven queries. No over-fetching. No under-fetching.

### How GraphQL Works

**Single endpoint:** `POST /graphql`

**Client sends a query describing exactly what it needs:**
```graphql
query {
  order(id: "o-789") {
    orderId
    status
    items {
      productId
      name
      price
      quantity
    }
    customer {
      name
      email
    }
  }
}
```

**Server returns exactly that shape — nothing more, nothing less:**
```json
{
  "data": {
    "order": {
      "orderId": "o-789",
      "status": "CONFIRMED",
      "items": [
        { "productId": "p-456", "name": "Nike Shoes", "price": 129.99, "quantity": 2 }
      ],
      "customer": {
        "name": "Vidhan",
        "email": "vidhan@example.com"
      }
    }
  }
}
```

No over-fetching — only the fields requested are returned.
No N+1 — order + items + customer in one round trip.

### Mutations — Writing Data

```graphql
mutation {
  createOrder(input: {
    userId: "u-123"
    items: [{ productId: "p-456", quantity: 2 }]
  }) {
    orderId
    status
    total
  }
}
```

### Subscriptions — Real-time Push

```graphql
subscription {
  orderStatusUpdated(orderId: "o-789") {
    status
    timestamp
  }
}
```
Runs over WebSocket — server pushes updates as they happen.

### GraphQL Problems:

**N+1 Query Problem:**
```
Query asks for 20 orders, each with customer name

Naive resolver:
  1 DB query for 20 orders
  20 DB queries for 20 customers (one per order)
  = 21 queries total

Solution: DataLoader — batches and caches resolver calls
  1 DB query for 20 orders
  1 DB query for all 20 customers (batched)
  = 2 queries total
```

**Complex query attacks:**
```graphql
# Malicious deeply nested query — can bring down your server
{
  orders {
    customer {
      orders {
        customer {
          orders {
            customer { ... }
          }
        }
      }
    }
  }
}
```
Solution: query depth limiting, query complexity scoring, rate limiting by query cost.

**Caching is harder:**
- REST GET requests cache naturally at HTTP layer (CDN, browser)
- GraphQL uses POST for queries — POST is not cached by default
- Requires application-level caching (persisted queries, Apollo Client cache)

### When to use GraphQL:
- Client-facing APIs with diverse clients (mobile, web, third-party) that need different data shapes
- Rapid product development where frontend needs change faster than backend
- When over-fetching and under-fetching are real problems (mobile on limited bandwidth)
- BFF (Backend For Frontend) layer aggregating multiple microservices

---

## Side-by-Side Comparison

| Feature | REST | gRPC | GraphQL |
|---|---|---|---|
| Transport | HTTP/1.1 or 2 | HTTP/2 only | HTTP/1.1 or 2 |
| Data format | JSON (text) | Protobuf (binary) | JSON (text) |
| Schema | Implicit / OpenAPI | Strongly typed `.proto` | Strongly typed SDL |
| Payload size | Large (JSON) | Small (binary) | Variable (client-driven) |
| Performance | Good | Excellent | Good |
| Browser support | Native | Needs gRPC-Web | Native |
| Streaming | Limited (SSE/WS) | Native (4 modes) | Via WebSocket |
| Caching | Excellent (HTTP) | Limited | Complex |
| Over/under-fetching | Yes | Yes | No |
| Learning curve | Low | Medium | Medium-High |
| Best for | Public APIs | Internal services | Complex client APIs |

---

## Real-World Example — ShopSphere Protocol Strategy

ShopSphere uses all three — at different layers:

```
External clients (mobile app, web frontend)
        ↓
   REST API (via API Gateway)
   ─ Standard, compatible, cacheable
   ─ GET /products, POST /orders, GET /orders/{id}

        ↓
   GraphQL BFF (Backend For Frontend)
   ─ Mobile app queries exactly the fields it needs
   ─ Aggregates data from multiple services in one request
   ─ Reduces mobile bandwidth and round trips

Internal microservice communication
        ↓
   gRPC
   ─ Order Service → Payment Service (high frequency, performance critical)
   ─ Order Service → Inventory Service (tight contract, generated client)
   ─ Notification Service streams order updates back to gateway
```

**This is a real pattern used at companies like Netflix, Uber, and Shopify** — REST or GraphQL at the edge for client compatibility, gRPC internally for performance and type safety.

---

## Interview Q&A

**Q: What is REST and what are its core constraints?**
REST is an architectural style for distributed systems built on HTTP. Its core constraints are statelessness — every request carries all needed context, resources identified by URLs, HTTP verbs carrying semantic intent, and a uniform interface. It is not a strict protocol — many APIs claim to be RESTful but violate these constraints.

**Q: What is the difference between REST and gRPC?**
REST uses HTTP with JSON and resource-based URLs — flexible, human-readable, universally compatible. gRPC uses HTTP/2 with Protobuf binary encoding and a schema-first RPC model — much faster, strongly typed, supports streaming, but not browser-native and requires shared proto contracts. REST is preferred for public APIs and external clients; gRPC is preferred for internal high-performance service-to-service communication.

**Q: What problem does GraphQL solve that REST cannot?**
REST endpoints return fixed data shapes — clients either receive too much data (over-fetching) or must make multiple requests to assemble what they need (under-fetching). GraphQL lets the client declare exactly what fields it needs in a single request, regardless of how many underlying services or entities are involved. This is especially valuable for mobile clients on limited bandwidth and for products where the frontend evolves faster than the backend.

**Q: What is the N+1 problem in GraphQL and how do you solve it?**
When resolving a list of items where each item requires a separate lookup — for example 20 orders each needing a customer name — a naive resolver makes 21 DB queries: one for orders, one per customer. The solution is DataLoader — it batches all customer lookups from a single resolver execution tick into a single DB query, then maps results back to the individual items.

**Q: Which protocol would you use for internal ShopSphere microservice communication and why?**
gRPC. Internal services communicate at high frequency over a controlled network where browser compatibility is not a concern. Protobuf binary encoding is 3–10x smaller than JSON and significantly faster to serialise, which matters at thousands of requests per second. The strongly typed proto contract enforced at compile time catches breaking changes before they reach production. Native streaming support in gRPC also handles use cases like real-time order status propagation without bolting on WebSockets.

