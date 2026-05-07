# Stage 2 — Topic 4: GraphQL (Deep Dive)

## Theory

We covered GraphQL's basics in Stage 1 Topic 11. Now we go deep — the execution model, the type system, the N+1 problem and DataLoader solution, schema design, security, and exactly where GraphQL fits in ShopSphere's architecture.

**The origin story matters here:**

Facebook built GraphQL in 2012 to solve a specific problem — their mobile app was on 2G networks, the REST API was returning massive JSON payloads full of unused data, and the iOS and Android apps needed completely different data shapes for the same screens. Every time the product team changed the UI, backend engineers had to build new endpoints or modify existing ones.

GraphQL flipped the model — **the client declares what it needs, the server delivers exactly that.** One endpoint. No versioning. Frontend teams move independently of backend teams.

---

## Internals — The GraphQL Execution Model

Understanding how GraphQL actually executes a query is what makes the N+1 problem and DataLoader make complete sense.

### The Type System — Schema Definition Language (SDL)

Everything in GraphQL starts with the schema. The schema is the contract — it defines every type, every field, every query, mutation, and subscription the server exposes:

```graphql
# Scalar types — leaf values
scalar DateTime
scalar Decimal

# Object types — structured data
type Order {
  id:           ID!              # ! means non-nullable — always present
  status:       OrderStatus!
  total:        Decimal!
  createdAt:    DateTime!
  customer:     User!            # nested object — resolver fetches this
  items:        [OrderItem!]!    # list of non-nullable items, list itself non-nullable
  shipment:     Shipment         # nullable — might not exist yet
}

type OrderItem {
  id:        ID!
  product:   Product!
  quantity:  Int!
  price:     Decimal!
}

type User {
  id:        ID!
  name:      String!
  email:     String!
  orders:    [Order!]!
}

type Product {
  id:          ID!
  name:        String!
  price:       Decimal!
  description: String
  category:    Category!
  inventory:   Int!
}

# Enum type
enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

# Entry points — how clients interact
type Query {
  order(id: ID!):          Order
  orders(filter: OrderFilter, pagination: PaginationInput): OrderConnection!
  product(id: ID!):        Product
  products(filter: ProductFilter): ProductConnection!
  me:                      User!
}

type Mutation {
  createOrder(input: CreateOrderInput!):   OrderPayload!
  cancelOrder(orderId: ID!):              OrderPayload!
  updateProfile(input: UpdateProfileInput!): UserPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): OrderUpdate!
}

# Input types — for mutations
input CreateOrderInput {
  items:             [OrderItemInput!]!
  paymentMethodId:   ID!
  shippingAddressId: ID!
  promoCode:         String
}

input OrderItemInput {
  productId: ID!
  quantity:  Int!
}

# Payload types — mutation responses
type OrderPayload {
  order:  Order
  errors: [UserError!]!
}

type UserError {
  field:   String
  message: String!
}
```

**Nullability is part of your contract:**
```graphql
name: String!   ← guaranteed to always be present — client can depend on it
name: String    ← might be null — client must handle null case

List nullability matters too:
[OrderItem!]!   ← list always present, items never null
[OrderItem]!    ← list always present, but items might be null
[OrderItem!]    ← list might be null, but if present items are not null
[OrderItem]     ← both list and items might be null
```

Design your schema with explicit nullability — it communicates guarantees to consumers.

---

### The Resolver Chain — How GraphQL Executes

This is the most important internal to understand. Every field in GraphQL is resolved by a **resolver function** — a function that knows how to fetch that specific piece of data.

```
Query:
{
  order(id: "o-789") {
    id
    status
    customer {
      name
      email
    }
    items {
      quantity
      product {
        name
        price
      }
    }
  }
}
```

GraphQL executes this as a **tree of resolver calls:**

```
Root Query resolver:
  order(id: "o-789")
    → SQL: SELECT * FROM orders WHERE id = 'o-789'
    → returns Order object
    
    ↓ field resolvers run on the returned Order object

  Order.id        → returns order.id       (trivial, no DB call)
  Order.status    → returns order.status   (trivial, no DB call)
  Order.customer  → SQL: SELECT * FROM users WHERE id = order.customerId
                  → returns User object
  
    ↓ field resolvers run on the returned User object
    
    User.name     → returns user.name      (trivial)
    User.email    → returns user.email     (trivial)

  Order.items     → SQL: SELECT * FROM order_items WHERE order_id = 'o-789'
                  → returns [OrderItem] array
  
    ↓ field resolvers run on EACH item in the array

    OrderItem[0].quantity  → trivial
    OrderItem[0].product   → SQL: SELECT * FROM products WHERE id = item.productId
    OrderItem[1].quantity  → trivial
    OrderItem[1].product   → SQL: SELECT * FROM products WHERE id = item.productId
    OrderItem[2].quantity  → trivial
    OrderItem[2].product   → SQL: SELECT * FROM products WHERE id = item.productId
```

**This is exactly how the N+1 problem emerges.** For a list of 20 orders each needing customer data — 1 query for orders + 20 queries for customers = 21 queries. The resolver architecture makes this the natural default.

---

## The N+1 Problem and DataLoader

### The Problem in Detail

```graphql
{
  orders {           # 1 DB query → returns 20 orders
    id
    status
    customer {       # 20 DB queries — one per order — THE N+1 PROBLEM
      name
    }
    items {          # 20 DB queries — one per order
      product {      # N queries per order × 20 orders — EXPONENTIAL
        name
        price
      }
    }
  }
}
```

For 20 orders with 5 items each:
```
1   query  for orders
20  queries for customers
20  queries for order items lists
100 queries for products (5 items × 20 orders)
────────────────────────
141 DB queries for one GraphQL request
```

This kills performance at scale.

### DataLoader — The Solution

DataLoader was built by Facebook specifically to solve this. It works on two principles:

**1. Batching — collect all IDs first, fetch in one query:**
```
Without DataLoader:
  t=0ms: SELECT * FROM users WHERE id = 'u-001'
  t=1ms: SELECT * FROM users WHERE id = 'u-002'
  t=2ms: SELECT * FROM users WHERE id = 'u-003'
  ...20 separate queries

With DataLoader:
  DataLoader collects all user IDs during the current execution tick
  Then fires ONE batched query:
  t=0ms: SELECT * FROM users WHERE id IN ('u-001','u-002','u-003',...,'u-020')
  Maps results back to individual resolvers
```

**2. Caching — within a single request, never fetch the same ID twice:**
```
Order 1 and Order 3 both belong to customer u-005
Without DataLoader: u-005 fetched twice
With DataLoader:    u-005 fetched once, second resolver gets cached result
```

**DataLoader implementation in Java (with Spring + GraphQL):**

```java
// Define a DataLoader for users
@Component
public class UserDataLoader implements BatchLoaderWithContext<String, User> {

    @Autowired
    private UserRepository userRepository;

    @Override
    public CompletionStage<List<User>> load(List<String> userIds, 
                                             BatchLoaderEnvironment env) {
        // One DB query for ALL user IDs
        List<User> users = userRepository.findAllById(userIds);
        
        // Must return in same order as input IDs
        Map<String, User> userMap = users.stream()
            .collect(Collectors.toMap(User::getId, u -> u));
        
        return CompletableFuture.completedFuture(
            userIds.stream()
                   .map(id -> userMap.getOrDefault(id, null))
                   .collect(Collectors.toList())
        );
    }
}

// Use DataLoader in resolver
@SchemaMapping(typeName = "Order", field = "customer")
public CompletableFuture<User> customer(Order order, DataLoader<String, User> userLoader) {
    // This does NOT fire a DB query immediately
    // DataLoader batches all customer IDs across all order resolvers
    // then fires ONE query for all of them
    return userLoader.load(order.getCustomerId());
}
```

**The execution with DataLoader:**
```
GraphQL resolves Order.customer for all 20 orders
  → Calls userLoader.load("u-001")   ← queued, not executed
  → Calls userLoader.load("u-002")   ← queued
  → ...
  → Calls userLoader.load("u-020")   ← queued

End of execution tick — DataLoader fires:
  SELECT * FROM users WHERE id IN ('u-001','u-002',...,'u-020')
  → 1 query instead of 20

Total queries for the same request:
1   query for orders
1   query for all customers (batched)
1   query for all order items (batched)
1   query for all products   (batched)
────────────────────
4 DB queries instead of 141
```

---

## Connections and Pagination — The Relay Pattern

GraphQL has a standard pagination pattern called **Relay Connections** — used by Facebook, GitHub, Shopify:

```graphql
type Query {
  orders(first: Int, after: String, last: Int, before: String): OrderConnection!
}

type OrderConnection {
  edges:    [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type OrderEdge {
  node:   Order!         # the actual order
  cursor: String!        # opaque cursor for this position
}

type PageInfo {
  hasNextPage:     Boolean!
  hasPreviousPage: Boolean!
  startCursor:     String
  endCursor:       String
}
```

**Client query:**
```graphql
{
  orders(first: 20, after: "cursor-abc") {
    edges {
      cursor
      node {
        id
        status
        total
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

**Why this pattern:**
- Cursors are stable — insertions don't cause duplicates or skips
- `edges` wrapper allows adding metadata per item (cursor, relevance score)
- `pageInfo` tells client exactly what to request next
- Forward and backward pagination both supported
- Industry standard — any GraphQL client library understands it

---

## Mutations — Design Patterns

**Input types — always use dedicated input types:**
```graphql
# Bad — scalar arguments directly
mutation {
  createOrder(userId: ID!, items: [ID!]!, paymentMethodId: ID!)
}

# Good — input type
mutation {
  createOrder(input: CreateOrderInput!): OrderPayload!
}

input CreateOrderInput {
  items:             [OrderItemInput!]!
  paymentMethodId:   ID!
  shippingAddressId: ID!
  promoCode:         String
}
```

Input types can be evolved — add optional fields without breaking existing mutations.

**Payload types — always return both data and errors:**
```graphql
type OrderPayload {
  order:  Order          # null if mutation failed
  errors: [UserError!]!  # empty array if success
}

type UserError {
  field:   String        # which field caused the error
  message: String!       # human-readable description
  code:    String        # machine-readable error code
}
```

**Why this pattern instead of throwing GraphQL errors:**
```graphql
# Client handles gracefully — no exception handling needed
mutation CreateOrder($input: CreateOrderInput!) {
  createOrder(input: $input) {
    order {
      id
      status
    }
    errors {
      field
      message
    }
  }
}

# Client code
if (data.createOrder.errors.length > 0) {
  showValidationErrors(data.createOrder.errors)
} else {
  showOrderConfirmation(data.createOrder.order)
}
```

---

## GraphQL Security

GraphQL's flexibility is also its attack surface. Production GraphQL must be hardened:

**1. Query Depth Limiting:**
```graphql
# Malicious query — infinite nesting crashes the server
{
  orders {
    customer {
      orders {
        customer {
          orders {
            customer { ... }  # 50 levels deep
          }
        }
      }
    }
  }
}
```
```java
// Spring for GraphQL — limit depth
GraphQlSource.schemaResourceBuilder()
    .instrumentation(new MaxQueryDepthInstrumentation(10))
```

**2. Query Complexity Scoring:**
```
Assign a cost to each field:
  Scalar field:  cost = 1
  Object field:  cost = 1
  List field:    cost = 10 (fetches many)
  
Total query cost must not exceed a threshold (e.g. 1000)

Complex attack query:
  orders (cost 10) × items (cost 10) × product (cost 1) × reviews (cost 10) = 1000+ → rejected
```

**3. Persisted Queries:**
```
Development: clients send arbitrary query strings
Production:  clients register queries with a hash during build

Client sends:  { "id": "abc123hash" }
Server looks up: "abc123hash" → stored query → executes
Unknown hash → rejected

Benefits:
  - Arbitrary query injection is impossible
  - Smaller payloads (hash vs full query string)
  - Easier whitelisting and auditing
```

**4. Rate Limiting by Query Cost:**
```
Instead of rate limiting by request count:
  100 requests/minute (standard)

Rate limit by complexity points:
  10,000 complexity points/minute

Simple queries (cost 10) → 1000 requests/minute
Complex queries (cost 500) → 20 requests/minute
```

**5. Disable Introspection in Production:**
```java
// Introspection lets anyone discover your entire schema
// Useful in development, dangerous in production
graphql:
  schema:
    introspection:
      enabled: false  # disable in prod
```

---

## Real-World Example — ShopSphere GraphQL BFF

ShopSphere uses GraphQL as a **BFF (Backend For Frontend)** layer — sitting between external clients and internal gRPC services:

```
Mobile App  ──► GraphQL BFF ──gRPC──► Order Service
Web App     ──►             ──gRPC──► Product Service
                            ──gRPC──► User Service
                            ──gRPC──► Inventory Service
```

**Why this architecture:**
- Mobile app needs different data shapes than web app — GraphQL handles both with one schema
- Reduces mobile bandwidth — app requests only the fields it renders
- Reduces round trips — one GraphQL query replaces 4–5 REST calls
- Internal services stay gRPC — fast, typed, streaming
- GraphQL BFF translates between client needs and internal service contracts

**A real mobile screen — order history with product thumbnails:**

```graphql
# Mobile sends one query
query OrderHistory($userId: ID!, $first: Int!) {
  me {
    orders(first: $first) {
      edges {
        node {
          id
          status
          createdAt
          total
          items {
            quantity
            product {
              name
              thumbnailUrl    # mobile only needs thumbnail, not full image list
              price
            }
          }
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

**BFF resolver translating to gRPC:**
```java
@SchemaMapping(typeName = "Query", field = "me")
public CompletableFuture<User> me(GraphQLContext context) {
    String userId = context.get("userId"); // from JWT
    
    // Call User Service via gRPC
    return userServiceStub.getUser(
        GetUserRequest.newBuilder().setUserId(userId).build()
    ).toCompletableFuture().thenApply(this::mapToGraphQLUser);
}

@SchemaMapping(typeName = "User", field = "orders")
public CompletableFuture<OrderConnection> orders(
        User user,
        @Argument int first,
        @Argument String after) {
    
    // Call Order Service via gRPC
    return orderServiceStub.listOrders(
        ListOrdersRequest.newBuilder()
            .setUserId(user.getId())
            .setPageSize(first)
            .setCursor(after != null ? after : "")
            .build()
    ).toCompletableFuture().thenApply(this::mapToConnection);
}
```

---

## Interview Q&A

**Q: What is the N+1 problem in GraphQL and how does DataLoader solve it?**
The N+1 problem occurs because GraphQL resolves each field independently. Fetching 20 orders where each needs a customer lookup triggers 1 query for orders and 20 separate queries for customers — 21 total. DataLoader solves this with batching and caching. It collects all customer IDs requested during a single execution tick, fires one batched `WHERE id IN (...)` query, then maps results back to individual resolvers. The same request goes from 21 queries to 2.

**Q: What is a GraphQL BFF and why would you use it?**
A BFF — Backend For Frontend — is a GraphQL layer sitting between clients and internal microservices. It aggregates data from multiple services into client-specific shapes in a single query. Mobile clients get only the fields they need for their screens, eliminating over-fetching on limited bandwidth. It decouples frontend teams from backend service contracts — UI changes don't require backend changes as long as the underlying data exists. Internal services stay as gRPC or REST, and the BFF translates between client needs and service contracts.

**Q: How do you prevent GraphQL from being abused with malicious queries?**
Four layers of defence — depth limiting prevents infinite nested queries from crashing the server, complexity scoring assigns a cost to each field and rejects queries exceeding a threshold, persisted queries allow only pre-registered query hashes in production making arbitrary query injection impossible, and rate limiting by complexity points penalises expensive queries more heavily than cheap ones. Introspection should also be disabled in production to prevent schema discovery.

**Q: What is the Relay Connection pattern and why use it over simple list pagination?**
Relay Connections wrap list results in an edges/node/cursor structure with a PageInfo object. Cursors are opaque and stable — unlike offset pagination, insertions don't cause duplicate or skipped items. The pageInfo provides hasNextPage and endCursor making client-side pagination logic trivial. It supports both forward and backward pagination. Being a GraphQL community standard, all major client libraries understand it natively — Apollo Client, Relay, URQL all handle it automatically.

**Q: When would you choose GraphQL over REST in a system design interview?**
GraphQL is the right choice when clients have diverse and evolving data needs — mobile apps needing minimal payloads, web apps needing richer data, third-party developers building unknown use cases. It eliminates over-fetching and under-fetching, removes the need for API versioning for additive changes, and lets frontend teams move independently of backend teams. The cost is complexity — DataLoader for N+1, security hardening against query attacks, and more complex caching compared to REST's HTTP-layer caching. For simple CRUD services with predictable access patterns, REST is simpler and sufficient.

---

Say **"next"** when ready for Topic 5 — Synchronous vs Asynchronous Communication.
