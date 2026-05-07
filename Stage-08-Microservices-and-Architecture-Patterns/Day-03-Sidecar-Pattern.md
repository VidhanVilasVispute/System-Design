## Topic 3 — Sidecar Pattern

### The Problem First

Every microservice needs more than just business logic. It also needs:

```
- Logging (structured, centralized)
- Metrics collection
- Distributed tracing
- mTLS (mutual TLS for secure service-to-service calls)
- Retries, timeouts, circuit breaking
- Auth token validation
```

If you bake all this into every service:

```
┌─────────────────────────────────────┐
│         Order Service               │
│                                     │
│  [Business Logic]                   │
│  [Retry Logic]                      │
│  [mTLS cert management]             │
│  [Metrics collection]               │
│  [Distributed trace propagation]    │
│  [Log formatting + shipping]        │
└─────────────────────────────────────┘

× Repeated in EVERY service
× Java team implements it differently than Go team
× Update retry logic → redeploy ALL 20 services
× Business logic buried under infrastructure code
```

This violates **separation of concerns** at scale.

---

### What is the Sidecar Pattern?

Deploy a **second process alongside your service** in the same pod/host. This co-located process handles all infrastructure concerns. Your service only does business logic.

```
┌─────────────────────────────────────────────────────┐
│                    K8s Pod                          │
│                                                     │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │   Order Service     │  │   Sidecar (Envoy)    │  │
│  │   (Your Code)       │  │                      │  │
│  │                     │  │  - mTLS              │  │
│  │  Business logic     │  │  - Retries           │  │
│  │  only               │  │  - Metrics           │  │
│  │                     │  │  - Tracing           │  │
│  │  Port 8080          │  │  - Access logs       │  │
│  └─────────────────────┘  └──────────────────────┘  │
│         ▲                          ▲                 │
│         │ localhost                │ all network I/O │
│         └──────────────────────────┘                 │
└─────────────────────────────────────────────────────┘
```

The sidecar and main container **share the same network namespace** — they see each other as `localhost`. All inbound and outbound traffic flows through the sidecar. Your service never knows it exists.

---

### Why "Sidecar"?

Named after the motorcycle sidecar — a separate attachment that rides alongside the main vehicle, sharing the same journey, but with a distinct role.

```
Motorcycle (your service) → does the driving (business logic)
Sidecar attachment        → carries the passenger (infra concerns)
Both travel together, both stop together
```

---

### How Traffic Flows

Without sidecar:
```
Client → Order Service (port 8080) → Product Service (port 8080)
```

With sidecar (Envoy/Istio):
```
Inbound:
  Client → Envoy (intercepts on port 15001) → Order Service (localhost:8080)

Outbound:
  Order Service (localhost) → Envoy (intercepts) → Product Service's Envoy → Product Service

Your service code is IDENTICAL. It still calls http://product-service:8080.
The sidecar transparently proxies everything.
```

---

### What the Sidecar Handles

#### 1. Observability

```
Every request in and out gets:
  - Logged automatically (method, path, latency, status code)
  - Traced (span created, traceId propagated to downstream calls)
  - Metrics emitted (request rate, error rate, p99 latency)

Your Java code has ZERO logging/tracing boilerplate for HTTP calls.
```

#### 2. mTLS (Mutual TLS)

```
Without sidecar:
  Order Service → plain HTTP → Product Service
  Anyone on the network can intercept or spoof requests

With Envoy sidecar:
  Order Envoy → TLS handshake (both sides verify certs) → Product Envoy → Product Service
  
  Your service speaks plain HTTP to localhost.
  Sidecar handles cert rotation, handshake, encryption.
  Zero cert management in your Java code.
```

#### 3. Retries and Circuit Breaking

```
Order Service calls Product Service.
Product Service returns 503.

Without sidecar: your Java code must implement retry logic.
With sidecar:    Envoy retries automatically (configurable: 3 attempts, 100ms backoff).
                 Your service sees a success — it never knew there was a failure.

Circuit breaker in Envoy:
  After 5 consecutive 503s → Envoy opens circuit
  → Returns error immediately without hitting Product Service
  → Protects Product Service from more load while it recovers
```

#### 4. Auth / JWT Validation

```
Request comes in with JWT token.
Sidecar validates the token before passing to your service.
If invalid → sidecar returns 401. Your service never called.

Your service can trust that any request reaching it is already authenticated.
```

---

### The Two Key Tools

#### Envoy Proxy

Open-source, high-performance L7 proxy written in C++. The foundational component of most service meshes. Originally built by Lyft.

```
Envoy capabilities:
  - HTTP/1.1, HTTP/2, gRPC proxying
  - Load balancing (round-robin, least-request, ring hash)
  - Circuit breaking
  - Retries with backoff
  - Rate limiting
  - Distributed tracing (generates spans, propagates traceId)
  - Access logging
  - mTLS termination

Configured via xDS API (dynamic config pushed from control plane)
→ You don't touch Envoy config directly in Istio.
  Istio's control plane pushes config to all Envoy sidecars.
```

#### Istio

A **service mesh** built on top of Envoy. Istio is the **control plane** — it manages all the Envoy sidecars across your cluster.

```
Istio Architecture:

  Control Plane (istiod):
  ┌────────────────────────────────────────┐
  │  - Pushes Envoy config via xDS API     │
  │  - Manages mTLS certificate lifecycle  │
  │  - Handles traffic policy rules        │
  │  - Collects telemetry metadata         │
  └────────────────────────────────────────┘
           │ pushes config to all sidecars
           ▼
  Data Plane (Envoy sidecars in every pod):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Order Pod │  │Product   │  │User Pod  │
  │[Envoy]   │  │Pod[Envoy]│  │[Envoy]   │
  │[Service] │  │[Service] │  │[Service] │
  └──────────┘  └──────────┘  └──────────┘
```

You define high-level rules in Istio (`VirtualService`, `DestinationRule`) and Istio translates them into Envoy config pushed to every sidecar automatically.

---

### Sidecar Injection in Kubernetes

In Istio, you don't manually add the Envoy container to every pod spec. Istio uses a **mutating admission webhook**:

```
You apply your normal Deployment YAML (single container).
Before K8s schedules the pod, Istio's webhook intercepts it.
Istio automatically injects the Envoy sidecar container.
The pod starts with 2 containers: your service + Envoy.

Enable per namespace:
  kubectl label namespace shopsphere istio-injection=enabled

Now every pod deployed to shopsphere namespace gets Envoy automatically.
Your Deployment YAML stays clean.
```

---

### Sidecar vs Library Approach

```
┌──────────────────┬───────────────────────┬─────────────────────────┐
│                  │  Library (Resilience4j│  Sidecar (Envoy/Istio)  │
│                  │  Micrometer, etc.)    │                         │
├──────────────────┼───────────────────────┼─────────────────────────┤
│ Language         │ Java/JVM only         │ Any language            │
│ Code coupling    │ In your codebase      │ Zero app code change    │
│ Update infra     │ Redeploy all services │ Update sidecar config   │
│ Debugging        │ Familiar (Java logs)  │ Extra layer to inspect  │
│ Performance      │ In-process (fast)     │ Extra network hop       │
│ Adoption         │ Per team choice       │ Enforced uniformly      │
└──────────────────┴───────────────────────┴─────────────────────────┘
```

In practice: **both are used together**. Resilience4j in your Java code for business-level fallbacks, Envoy sidecar for infra-level retries and observability.

---

### ShopSphere Lens

```
Right now, ShopSphere uses:
  - Micrometer + Actuator for metrics (library approach)
  - Manual MDC for correlation IDs in logs (library approach)
  - Resilience4j for circuit breaking (library approach)

When ShopSphere moves to full K8s + Istio:
  - Envoy sidecar auto-injected into every service pod
  - mTLS between all 22 services — zero cert code in Java
  - Retries/timeouts moved to Istio VirtualService YAML
  - Distributed tracing spans generated by Envoy, not your code
  - Your Feign clients still call http://product-service:8080
    → Envoy intercepts transparently

The ShopSphere Java code gets SIMPLER as Istio handles more.
```

---

### Interview Questions

**Q1. What problem does the Sidecar Pattern solve?**

> Cross-cutting infrastructure concerns (logging, tracing, mTLS, retries) would otherwise be duplicated across every service. The sidecar externalizes these into a co-located proxy process, keeping service code focused on business logic and making infrastructure concerns language-agnostic and centrally configurable.

**Q2. How does Envoy intercept traffic transparently without the app knowing?**

> Istio uses **iptables rules** (set up by an init container called `istio-init`) to redirect all inbound and outbound traffic to Envoy's ports. The app still binds to port 8080 and calls other services normally. The kernel-level redirect is invisible to the application.

**Q3. What's the difference between Envoy and Istio?**

> Envoy is the **data plane** — the actual proxy that handles traffic. Istio is the **control plane** — it manages, configures, and coordinates all Envoy sidecars across the cluster. Envoy can be used standalone, but Istio makes it manageable at scale by pushing config centrally.

**Q4. What's the downside of the Sidecar Pattern?**

> Every pod now has two containers instead of one — higher **resource overhead** (each Envoy uses ~50MB RAM). Every request makes an extra **localhost network hop** through Envoy — adds ~1–5ms latency. Also, debugging is harder since there's now an extra layer between client and service. For very latency-sensitive services, this tradeoff matters.

---

Ready for **Topic 4 — API Composition** when you are.
