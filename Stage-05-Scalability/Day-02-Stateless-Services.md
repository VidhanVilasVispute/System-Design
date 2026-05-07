# Topic 2 — Stateless Services

## What Does "State" Even Mean?

Before stateless vs stateful — let's be precise about what **state** is in a server context.

```
State = any data that must persist between two separate requests
        from the same client, stored on the server side.

Examples of state:
  - "User #42 is logged in" (session)
  - "User #42 has 3 items in cart" (cart data)
  - "User #42's last search was 'Nike shoes'" (search context)
  - "This request is part of a multi-step checkout flow" (workflow state)
```

---

## Stateful Server — The Problem

```
Traditional Web App (2005-era):

  User logs in → server creates a session object in memory

  ┌─────────────────────────────────────────┐
  │           App Server Memory             │
  │  sessions = {                           │
  │    "sess_abc123": { userId: 42,         │
  │                     cart: [...],        │
  │                     loggedIn: true }    │
  │    "sess_xyz789": { userId: 99, ... }   │
  │  }                                      │
  └─────────────────────────────────────────┘

  Browser stores only: Cookie → JSESSIONID=sess_abc123
```

Now you try to scale horizontally:

```
Request 1: POST /login ────────────────▶ svc-1 (session created here)
Request 2: GET  /cart  ──────────────┐
                                     ├──▶ svc-2 (session NOT here) → 401 ❌
Request 3: POST /checkout ───────────┘
```

The user experience breaks. Requests must **always** go to the same server. This is called **sticky sessions** and it's a band-aid, not a fix.

### Sticky Sessions — why they're a trap

```
Load Balancer with sticky sessions:

Client A ──always──▶ svc-1
Client B ──always──▶ svc-2
Client C ──always──▶ svc-1

Problem 1: svc-1 dies → Client A and C lose sessions → forced logout
Problem 2: svc-1 gets 80% traffic, svc-2 gets 20% → uneven load
Problem 3: deploying svc-1 forces active users offline
Problem 4: can't scale down svc-1 while sessions are active
```

Sticky sessions are **vertical scaling in disguise**. You've just made each server a mini-monolith for its subset of users.

---

## Stateless Server — The Solution

A stateless server holds **zero client-specific data in memory between requests**.

```
Stateless Rule:
  Every request must contain ALL information needed to process it.
  The server is just a function: f(request) → response

  No memory of past requests.
  No session objects.
  No in-process cache of user data.
```

```
┌──────────┐    Request carries identity + context     ┌──────────┐
│  Client  │ ─────────────────────────────────────────▶│  svc-1   │
│          │  Authorization: Bearer <JWT>               │          │
│          │  Body: { orderId: 99, ... }                │  reads   │
│          │                                            │  from    │──▶ DB/Redis
└──────────┘                                            │  shared  │
                                                        │  store   │
                                                        └──────────┘

Next request can go to ANY instance — they're all identical.
```

---

## How JWT Makes Auth Stateless — Deep Dive

This is the most important real-world implementation. ShopSphere already uses this.

### The old way (stateful session):
```
Login:
  1. User sends credentials
  2. Server validates, creates session: { userId: 42, roles: ["USER"] }
  3. Stores session in memory (or DB)
  4. Returns session ID cookie

Every request:
  1. Browser sends JSESSIONID=sess_abc123
  2. Server looks up session in memory/DB
  3. DB hit on every single request ← kills performance at scale
```

### The JWT way (stateless):
```
Login:
  1. User sends credentials
  2. Server validates, creates JWT:

  Header:  { alg: "HS256", typ: "JWT" }
  Payload: { sub: "42", roles: ["USER"], exp: 1710000000, iat: 1709996400 }
  Signature: HMAC_SHA256(base64(header) + "." + base64(payload), SECRET_KEY)

  JWT = base64(header).base64(payload).signature
  3. Returns JWT to client — server stores NOTHING

Every request:
  1. Client sends: Authorization: Bearer <JWT>
  2. Server verifies signature using SECRET_KEY (pure CPU operation, no DB)
  3. Decodes payload → gets userId, roles instantly
  4. Processes request

No DB hit. No shared memory. Any instance can verify any JWT.
```

```
Why can't someone forge a JWT?

  JWT payload is base64, not encrypted — anyone can read it.
  But to CHANGE it (e.g. change userId from 42 to 1 = admin),
  you must recompute the signature.
  You can't recompute it without the SECRET_KEY, which only servers know.
  Tampered JWT → signature mismatch → rejected.
```

### JWT in ShopSphere's request flow:
```
Client
  │
  │  GET /api/orders  +  Authorization: Bearer eyJhbGc...
  ▼
API Gateway (Spring Cloud Gateway)
  │  Validates JWT signature
  │  Extracts userId, roles from payload
  │  Adds headers: X-User-Id: 42, X-User-Roles: USER
  ▼
order-service
  │  Trusts X-User-Id header (internal network)
  │  No JWT re-validation needed
  │  No session lookup
  ▼
PostgreSQL
```

---

## Where Does State Actually Live Then?

If the server holds nothing, state must live in **external shared stores**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    What needs state?                            │
├────────────────────┬────────────────────────────────────────────┤
│  Auth / Identity   │  JWT (client carries it) — no server store │
│  User sessions     │  Redis (TTL-based, shared across instances)│
│  Shopping cart     │  Redis or DB (keyed by userId)             │
│  Order data        │  PostgreSQL                                │
│  Rate limit count  │  Redis (atomic increment)                  │
│  Feature flags     │  Redis or config service                   │
│  File uploads      │  S3 / object storage                       │
│  Async jobs        │  Kafka / RabbitMQ queue                    │
└────────────────────┴────────────────────────────────────────────┘

The app instances themselves: pure logic, zero state.
```

```
The Golden Rule of Stateless Architecture:

  Shared state → External store (Redis, DB, S3, Queue)
  Per-request context → Carried in the request itself (JWT, headers)
  App server → Stateless compute only
```

---

## Stateless vs Stateless — Common Misconceptions

**Misconception 1: "Stateless means no database"**
```
Wrong. Stateless means the SERVER INSTANCE has no state.
The database absolutely holds state — it's the single source of truth.
That's the entire point: centralize state in dedicated stores,
not distributed across random app instances.
```

**Misconception 2: "In-memory cache is fine"**
```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  svc-1   │     │  svc-2   │     │  svc-3   │
│          │     │          │     │          │
│ local    │     │ local    │     │ local    │
│ cache:   │     │ cache:   │     │ cache:   │
│ product  │     │ product  │     │ product  │
│ #5 =     │     │ #5 =     │     │ #5 =     │
│ ₹999     │     │ ₹1299 ←  │     │ ₹999     │
└──────────┘     └──────────┘     └──────────┘
                      ↑
               stale after price update!

Local in-memory cache = different instances see different data = bugs
Fix: Use Redis (shared, all instances read same data)
```

**Misconception 3: "Stateless is always better"**
```
Not always. Some systems NEED server-side state:
- WebSocket connections (persistent, stateful by nature)
- Long-running computations mid-execution
- Session-heavy legacy apps

For these, you use consistent hashing or sticky routing
deliberately — not accidentally.
```

---

## The 12-Factor App Principle

The industry standard for stateless, horizontally scalable services. Factor VI directly addresses this:

```
12-Factor App — Factor VI: Processes

  "Execute the app as one or more stateless processes.
   Any data that needs to persist must be stored in
   a stateful backing service (database, Redis, etc.)

   The memory space or filesystem of the process can
   be used as a brief, single-transaction cache.
   The app never assumes anything cached in memory
   will be available for future requests."
```

ShopSphere follows this by design — every service is a stateless Spring Boot process, all state externalized.

---

## Designing Stateless: Checklist

When building or reviewing a service, ask:

```
✅ Does the service store anything in a local HashMap / List between requests?
✅ Does it use HttpSession?
✅ Does it have instance-level fields that change at runtime?
✅ Does it write temp files to local disk expected by the next request?
✅ Does a background thread write to memory that request threads read?

If YES to any → service is NOT safely horizontally scalable.
Fix: move that state to Redis, DB, or the message queue.
```

---

## ShopSphere Audit

```
Service            Stateless?   Why
─────────────────────────────────────────────────────────────
user-service       ✅           JWT-based auth, no HttpSession
order-service      ✅           All state in PostgreSQL
product-service    ✅           Redis for cache (shared), no local cache
search-service     ✅           Elasticsearch holds all index state
notification-svc   ✅           RabbitMQ is the state (message queue)
review-service     ✅           PostgreSQL-backed
api-gateway        ✅           Validates JWT, routes, no session

One thing to watch: If you ever add a WebSocket feature
(e.g. order status push), that service needs sticky routing
or a pub/sub layer (Redis pub/sub) so any instance can
push to the right client.
```

---

## Interview Angles

**Q: What does it mean for a service to be stateless?**
> No client-specific data is stored in the server's memory between requests. Every request carries all the context needed to process it. All persistent state lives in external stores — databases, Redis, message queues.

**Q: How does JWT enable stateless authentication?**
> JWT encodes the user's identity and claims in a self-contained, cryptographically signed token. The server verifies the signature on every request without any DB lookup. No session table, no in-memory session map. Any instance can verify any JWT.

**Q: What's the difference between stateless app and stateless system?**
> The app tier is stateless — instances hold no client data. But the system as a whole has state, centralized in dedicated stores (PostgreSQL, Redis). Stateless doesn't mean no data — it means no data *on the app instances themselves*.

**Q: What breaks if you use local in-memory cache in a horizontally scaled service?**
> Cache inconsistency. Different instances will have different cached values for the same key. After a price update, svc-1 might serve ₹999 and svc-2 ₹1299 for the same product. Fix: shared Redis cache — one source of truth.

---

## Summary

```
┌───────────────────┬────────────────────────────────────────────────────┐
│  Stateful         │  Session in server memory, sticky sessions,        │
│                   │  breaks horizontal scaling, SPOF per user          │
├───────────────────┼────────────────────────────────────────────────────┤
│  Stateless        │  Zero server memory between requests,              │
│                   │  client carries identity (JWT),                    │
│                   │  shared state in Redis/DB,                         │
│                   │  any instance = any request ✅                     │
├───────────────────┼────────────────────────────────────────────────────┤
│  State lives in   │  DB (persistent), Redis (fast/TTL),                │
│                   │  Queue (async), Client (JWT/cookie)                │
├───────────────────┼────────────────────────────────────────────────────┤
│  ShopSphere       │  All services stateless by design.                 │
│                   │  JWT via Gateway → X-User-Id header forwarding.    │
│                   │  Redis for shared cache. PostgreSQL for truth.     │
└───────────────────┴────────────────────────────────────────────────────┘
```

---

Topics 1 and 2 are the foundation everything else in Stage 5 builds on. Next up is **Topic 3 — Rate Limiting**, where we go deep on token bucket, sliding window, fixed window, and how to do it *distributed* across multiple instances. Ready?
