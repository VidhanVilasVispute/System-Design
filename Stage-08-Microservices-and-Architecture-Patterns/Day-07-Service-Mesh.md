## Topic 7 — Service Mesh

### The Problem First

As ShopSphere grows to 22 services, every service needs:

```
- Retries + timeouts on every outbound call
- mTLS between every pair of services
- Circuit breakers for every downstream dependency
- Metrics: request rate, error rate, latency per service pair
- Distributed tracing for every request
- Access control: can order-service call payment-service?
```

Without a service mesh, you implement all this **inside each service**:

```
order-service:
  Resilience4j config for product-service  ✓
  Resilience4j config for payment-service  ✓
  Resilience4j config for user-service     ✓
  mTLS cert handling                       ✓
  Micrometer metrics per downstream        ✓

× Repeated across all 22 services
× Java teams implement it, Go team implements it differently
× Change retry policy → redeploy all 22 services
× mTLS cert rotation → code change in every service
```

A **Service Mesh** moves ALL of this out of your code into the infrastructure layer.

---

### What is a Service Mesh?

A dedicated infrastructure layer that handles **all service-to-service communication** transparently, via sidecar proxies injected into every pod.

```
WITHOUT Service Mesh:
┌─────────────────────────────────┐
│ Order Service                   │
│  [Business Logic]               │
│  [Retry Logic]                  │
│  [mTLS]                         │
│  [Metrics]                      │
│  [Circuit Breaker]              │
└─────────────────────────────────┘
        │ HTTP
        ▼
┌─────────────────────────────────┐
│ Payment Service                 │
│  [Business Logic]               │
│  [mTLS]                         │
│  [Metrics]                      │
└─────────────────────────────────┘

WITH Service Mesh:
┌──────────────────────────────────────────┐
│ Pod                                      │
│  ┌─────────────┐    ┌─────────────────┐  │
│  │Order Service│◀──▶│  Envoy Sidecar  │  │
│  │[Biz Logic   │    │  retries        │  │
│  │ ONLY]       │    │  mTLS           │  │
│  └─────────────┘    │  metrics        │  │
│                     │  circuit break  │  │
│                     └────────┬────────┘  │
└──────────────────────────────┼───────────┘
                               │ mTLS
                               ▼
┌──────────────────────────────────────────┐
│ Pod                                      │
│  ┌─────────────┐    ┌─────────────────┐  │
│  │Payment Svc  │◀──▶│  Envoy Sidecar  │  │
│  │[Biz Logic   │    │  mTLS           │  │
│  │ ONLY]       │    │  metrics        │  │
│  └─────────────┘    └─────────────────┘  │
└──────────────────────────────────────────┘
```

Your Java code shrinks. Infrastructure concerns move to the mesh.

---

### Architecture — Control Plane vs Data Plane

This is the most important concept to understand clearly:

```
┌─────────────────────────────────────────────────────────┐
│                  CONTROL PLANE (istiod)                 │
│                                                         │
│  ┌────────────┐  ┌──────────┐  ┌──────────────────────┐ │
│  │   Pilot    │  │  Citadel │  │       Galley         │ │
│  │ traffic    │  │  certs   │  │  config validation   │ │
│  │ management │  │  mTLS    │  │                      │ │
│  └─────┬──────┘  └────┬─────┘  └──────────────────────┘ │
│        │              │                                  │
│        └──────────────┴─── pushes config via xDS API ───┤
└─────────────────────────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Envoy Sidecar│  │ Envoy Sidecar│  │ Envoy Sidecar│
│ order-service│  │ product-svc  │  │ payment-svc  │
└──────────────┘  └──────────────┘  └──────────────┘
         DATA PLANE — handles actual traffic
```

**Control Plane (istiod):**
- Knows about all services in the mesh
- Manages and distributes TLS certificates
- Translates your high-level policy rules into Envoy config
- Pushes config to every Envoy sidecar dynamically — no restart needed

**Data Plane (Envoy sidecars):**
- Handles actual request traffic
- Enforces the policies pushed by control plane
- Emits metrics and traces
- Does the actual mTLS handshakes

---

### What a Service Mesh Gives You

#### 1. Automatic mTLS

```
Without mesh:
  order-service → plain HTTP → payment-service
  Anyone on cluster network can intercept

With Istio:
  Istiod issues certificates to every service automatically
  Envoy-to-Envoy communication is always mTLS encrypted
  Certificate rotation happens automatically every 24 hours
  Your Java code: still calls http://payment-service:8080
  The sidecar upgrades it to mTLS transparently

PeerAuthentication policy — enforce mTLS cluster-wide:
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: default
    namespace: shopsphere
  spec:
    mtls:
      mode: STRICT   ← reject any non-mTLS traffic
```

---

#### 2. Traffic Management — VirtualService

Fine-grained control over how traffic is routed, without changing code:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
    - product-service
  http:
    - timeout: 3s              # global timeout
      retries:
        attempts: 3            # retry up to 3 times
        perTryTimeout: 1s      # each attempt max 1s
        retryOn: 5xx,reset     # retry on server errors
      route:
        - destination:
            host: product-service
```

This applies to **every service** calling product-service — uniformly, no code changes.

---

#### 3. Traffic Splitting — Canary via Mesh

```yaml
# Send 90% to v1, 10% to v2 of product-service
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
    - product-service
  http:
    - route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
```

No load balancer config change. No code change. Pure YAML.

---

#### 4. Circuit Breaking — DestinationRule

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5       # after 5 consecutive 5xx
      interval: 30s                  # measured over 30s window
      baseEjectionTime: 30s          # eject for 30s
      maxEjectionPercent: 50         # never eject >50% of pods
```

Istio automatically stops sending traffic to unhealthy pods and reinstates them after the ejection window.

---

#### 5. Observability — For Free

Every sidecar automatically emits:

```
Metrics (to Prometheus):
  istio_requests_total{source="order-service", dest="payment-service", code="200"}
  istio_request_duration_milliseconds{...}
  istio_request_bytes{...}

Traces (to Jaeger):
  Every request gets spans created at sidecar level
  Even if your Java code has no OTel instrumentation

Access logs:
  [2024-01-01 10:00:00] order-service → payment-service
  POST /payments 200 245ms

You get golden signals (traffic, errors, latency, saturation)
for every service pair with ZERO code instrumentation.
```

---

#### 6. Authorization Policy

Control which services can talk to which:

```yaml
# Only order-service can call payment-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: shopsphere
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/shopsphere/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments/*"]
```

If notification-service tries to call payment-service directly — **denied at the sidecar level**, never reaches your Java code.

---

### Istio vs Linkerd

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│                  │ Istio                │ Linkerd              │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Proxy            │ Envoy (C++)          │ linkerd2-proxy (Rust)│
│ Resource usage   │ Higher (~50MB/sidecar│ Lower (~10MB/sidecar)│
│ Features         │ Very rich            │ Focused/simpler      │
│ Learning curve   │ Steep                │ Gentler              │
│ Traffic mgmt     │ Very powerful        │ Basic                │
│ mTLS             │ Yes                  │ Yes (automatic)      │
│ Observability    │ Excellent            │ Good                 │
│ Maturity         │ CNCF Graduated       │ CNCF Graduated       │
│ Best for         │ Complex requirements │ Simplicity first     │
└──────────────────┴──────────────────────┴──────────────────────┘

For ShopSphere at scale: Istio
For smaller teams wanting quick win: Linkerd
```

---

### When NOT to Use a Service Mesh

```
Service mesh adds real costs:
  - Every pod now has 2 containers (resource overhead)
  - ~1-5ms latency per hop (extra proxy in path)
  - Operational complexity: new CRDs, new failure modes
  - Debugging harder: is this a service bug or a mesh config bug?
  - Steep learning curve for the team

NOT worth it if:
  - Small team, fewer than 5-10 services
  - No multi-language stack (all Java → use Resilience4j library)
  - No strict mTLS requirement
  - Team unfamiliar with Kubernetes already

Worth it if:
  - 10+ services, multiple language runtimes
  - Strict zero-trust security requirements
  - Need fine-grained traffic control (canary, A/B testing)
  - Central policy enforcement across all services
```

---

### ShopSphere Lens

```
ShopSphere today (Docker Compose, 22 services):
  Resilience4j in Java code → handles circuit breaking
  Manual MDC → handles correlation
  Micrometer → handles metrics
  No mTLS (internal Docker network, acceptable for dev)

ShopSphere in production K8s + Istio:
  1. Label namespace: kubectl label ns shopsphere istio-injection=enabled
  2. Redeploy all services → Envoy auto-injected into every pod
  3. Apply PeerAuthentication STRICT → automatic mTLS everywhere
  4. Apply VirtualService for timeouts/retries → remove Resilience4j HTTP configs
  5. Apply AuthorizationPolicy → only valid service-to-service calls allowed
  6. Prometheus + Grafana + Jaeger → full observability with zero app changes

Your Java code becomes:
  - Business logic
  - Domain events (Kafka)
  - DB queries (JPA)
  → Infrastructure concerns fully delegated to the mesh
```

---

### Interview Questions

**Q1. What is a service mesh and what problem does it solve?**

> A service mesh is an infrastructure layer that handles all service-to-service communication concerns — mTLS, retries, circuit breaking, observability — via sidecar proxies co-located with each service. It solves the problem of duplicating cross-cutting infrastructure logic across every service in every language, by moving those concerns out of application code into a uniformly managed proxy layer.

**Q2. What is the difference between the control plane and data plane in Istio?**

> The data plane (Envoy sidecars) handles actual request traffic — enforcing policies, doing mTLS, emitting metrics. The control plane (istiod) manages the sidecars — it distributes TLS certificates, translates high-level policy rules into Envoy configuration, and pushes that config to all sidecars dynamically via the xDS API. You interact with the control plane; it takes care of the data plane.

**Q3. How does Istio enforce mTLS without any changes to application code?**

> Istio's control plane (Citadel component inside istiod) issues short-lived X.509 certificates to every service via the sidecar. The Envoy sidecar intercepts all outbound traffic using iptables rules, performs the mTLS handshake with the destination's sidecar, and forwards the decrypted request to the local service on localhost. The application only ever sees plain HTTP on localhost — the encryption is fully transparent.

**Q4. Service mesh vs library approach (Resilience4j + Micrometer) — how do you choose?**

> Libraries are appropriate for homogeneous stacks (all JVM), smaller service counts, and teams that want familiar debugging tools. A service mesh is appropriate when you have polyglot services, need centrally enforced policy (zero-trust security, uniform retry rules), or want to remove infrastructure concerns from application code entirely. In practice, production systems often use both — the mesh handles network-level concerns while application libraries handle business-level fallback logic.

---

Ready for **Topic 8 — Blue-Green Deployments** when you are.
