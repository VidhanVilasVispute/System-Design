

## Topic 1: Load Balancers

## Theory

As soon as your system handles more traffic than a single server can manage, you need to distribute that traffic across multiple servers. A **load balancer** is the component that does this — it sits in front of a pool of servers and routes each incoming request to one of them.

But a load balancer is more than a traffic distributor. It is also a **health monitor**, a **failover mechanism**, and in its more advanced form, an **application-aware router**. Understanding it at this depth is what separates a surface-level answer from a strong one.

**The core problem it solves:**

```
Without load balancer:
  All traffic → one server
  Server gets overwhelmed → slow responses → crashes
  Server goes down → entire service unavailable
  Scaling means pointing DNS to new server → downtime

With load balancer:
  All traffic → load balancer → distributed across N servers
  One server crashes → load balancer detects, stops routing to it
  Add a new server → load balancer detects, starts routing to it
  Zero downtime scaling and failover
```

---

## L4 vs L7 — The Most Important Distinction

Load balancers operate at different layers of the network stack. The two that matter are L4 and L7.

### L4 — Transport Layer Load Balancer

Operates at the TCP/UDP level. It sees:
- Source IP and port
- Destination IP and port
- Protocol (TCP or UDP)

It does **not** see HTTP headers, URLs, cookies, or request bodies. It has no idea what kind of traffic it is handling — it just moves bytes.

```
Client: 192.168.1.5:54231
    │
    ▼
L4 Load Balancer
    │  Only sees: IP addresses + ports
    │  Does NOT see: /api/orders, Authorization header, body
    │
    ├──► Server 1: 10.0.0.1:8080
    ├──► Server 2: 10.0.0.2:8080
    └──► Server 3: 10.0.0.3:8080
```

**How L4 balancing works:**
```
TCP connection arrives from client
L4 LB picks a backend server (by algorithm)
Creates a NEW TCP connection to that backend
Forwards all bytes from client connection to backend connection
Acts as a transparent TCP proxy — client does not know about the backend

Performance:
  Extremely fast — no packet inspection
  Millions of connections per second
  Minimal latency overhead (~microseconds)
  Low CPU usage
```

**L4 limitations:**
```
Cannot route based on URL path
Cannot route based on HTTP headers
Cannot do SSL termination (sees encrypted bytes only)
Cannot make intelligent routing decisions based on content
Sticky sessions only by IP — not by cookie or session ID
```

**When L4 is the right choice:**
- Non-HTTP traffic — database connections (PostgreSQL, MySQL), raw TCP services
- Extremely high throughput where L7 overhead is unacceptable
- Simple even distribution with no content-based routing needed
- UDP load balancing (DNS, game servers, video streaming)

---

### L7 — Application Layer Load Balancer

Operates at the HTTP/HTTPS level. It sees everything:
- Full HTTP request — method, URL, headers, body
- Cookies and session information
- SSL/TLS (can terminate it)
- Response status codes and bodies

```
Client Request:
  GET /api/v1/orders/o-789
  Host: api.shopsphere.com
  Authorization: Bearer eyJh...
  Cookie: session=abc123
    │
    ▼
L7 Load Balancer
    │  Sees EVERYTHING — can make intelligent decisions
    │
    │  Path /api/v1/orders/**  → Route to Order Service pool
    │  Path /api/v1/products/**→ Route to Product Service pool
    │  Header: API-Version: v2 → Route to v2 instances
    │  Cookie: beta=true       → Route to canary instances
    │
    ├──► Order Service instances
    ├──► Product Service instances
    └──► Search Service instances
```

**L7 capabilities:**
```
Content-based routing:
  /api/orders/**    → order service pool
  /api/products/**  → product service pool
  /static/**        → CDN or static file servers

SSL termination:
  Decrypts HTTPS at the load balancer
  Forwards plain HTTP to backends
  Backends don't need SSL certificates
  Centralised certificate management

Sticky sessions by cookie:
  Insert a cookie identifying the backend server
  All requests with that cookie go to the same server
  More reliable than IP-based stickiness

Request/response manipulation:
  Add headers before forwarding (X-Forwarded-For)
  Remove sensitive headers from responses
  Inject tracing headers

Health checks:
  HTTP health checks — GET /health → expect 200
  Understands application health, not just TCP connectivity
  A server that accepts TCP but returns 500s can be removed

Compression:
  Compress responses before sending to clients

Authentication offloading:
  Some L7 LBs can validate JWTs
  Reject unauthenticated requests before they reach services
```

---

## Load Balancing Algorithms

The algorithm determines which backend server gets each request. This is more nuanced than it looks.

### Round Robin

```
Requests:  1    2    3    4    5    6
Servers:   A    B    C    A    B    C

Each server gets an equal share in rotation.
Simple, fair for equal-capacity servers.
Problem: ignores actual server load.
```

**Weighted Round Robin:**
```
Server A: weight 3 (handles 3x the load)
Server B: weight 1
Server C: weight 2

Distribution: A A A B C C A A A B C C...

Use when servers have different capacities.
New server joins with lower weight during warm-up.
```

### Least Connections

```
Request arrives → route to server with FEWEST active connections

Current state:
  Server A: 50 active connections
  Server B: 23 active connections  ← route here
  Server C: 41 active connections

Best for long-lived connections where request duration varies.
Database connections, file uploads, streaming.
Prevents routing new requests to an already-busy server.
```

**Weighted Least Connections:**
```
Divide connections by weight to normalise:
  Server A: 50 connections / weight 4 = 12.5
  Server B: 23 connections / weight 2 = 11.5  ← route here
  Server C: 41 connections / weight 3 = 13.7

Combines server capacity with current load.
Most accurate for heterogeneous server pools.
```

### IP Hash

```
hash(client_IP) % number_of_servers = server_index

Client 192.168.1.5 → hash → always routes to Server B
Client 192.168.1.6 → hash → always routes to Server A

Provides sticky sessions without cookies.
Same client always hits same server.

Problem: if a server goes down, all its clients
         are redistributed — breaks sessions.
Problem: poor distribution if many clients behind one NAT IP.
Better alternative: consistent hashing (Stage 3 Topic 5)
```

### Least Response Time

```
Track average response time per backend server.
Route to the server with lowest (connections × response_time).

Most sophisticated — accounts for both load and performance.
Used by Nginx Plus, AWS ALB.
Best for APIs where response time varies significantly.
```

### Random with Two Choices (Power of Two)

```
Instead of evaluating all servers:
  Pick 2 random servers
  Send to the one with fewer connections

Surprisingly close to optimal least-connections
at a fraction of the coordination cost.
Scales extremely well in distributed systems.
Used by Netflix Ribbon, Envoy proxy.
```

---

## Health Checks — How Load Balancers Know a Server is Down

Without health checks, a load balancer blindly routes to dead servers. Health checks are critical:

### Passive Health Checks

```
Load balancer observes responses passively.
If a server returns 5xx errors or times out:
  → Mark server as unhealthy
  → Stop routing to it
  → Remove from rotation

Problem: real requests fail while detecting the problem.
         Users see errors during detection window.
```

### Active Health Checks

```
Load balancer proactively sends health check requests:
  Every 5 seconds:
    GET http://server-1:8080/actuator/health
    Expected: 200 OK { "status": "UP" }

If health check fails:
  → Server marked unhealthy
  → Removed from rotation immediately
  → No real requests affected

If health check recovers:
  → Server marked healthy after N consecutive successes
  → Added back to rotation gradually
```

**Health check endpoint in Spring Boot (ShopSphere):**
```java
// Spring Actuator provides this automatically
// GET /actuator/health
// Response:
{
  "status": "UP",
  "components": {
    "db": { "status": "UP" },          // DB connection healthy
    "redis": { "status": "UP" },       // Redis connection healthy
    "kafka": { "status": "UP" },       // Kafka connection healthy
    "diskSpace": { "status": "UP" }
  }
}

// Custom health indicator
@Component
public class OrderServiceHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // Check critical dependency
        boolean canConnectToPaymentService = paymentServiceClient.ping();
        if (!canConnectToPaymentService) {
            return Health.down()
                .withDetail("reason", "Cannot reach payment service")
                .build();
        }
        return Health.up().build();
    }
}
```

**Liveness vs Readiness (Kubernetes):**
```
Liveness probe:
  Is the service alive? (not deadlocked or crashed)
  Failure → container is restarted
  GET /actuator/health/liveness

Readiness probe:
  Is the service ready to receive traffic?
  Failure → removed from load balancer rotation but NOT restarted
  GET /actuator/health/readiness
  
  Use case: service is alive but warming up cache,
  running DB migrations, connecting to dependencies.
  Should not receive traffic yet but should not be restarted.
```

---

## SSL Termination

L7 load balancers handle SSL termination — decrypting HTTPS so backends only deal with HTTP:

```
Client ──HTTPS──► Load Balancer ──HTTP──► Backend Servers

Benefits:
  ✅ Centralised certificate management (one cert, one place)
  ✅ Backend servers no longer need SSL handling — lower CPU usage
  ✅ Load balancer can inspect request content (L7 routing)
  ✅ Certificate renewal in one place — no per-service cert rotation

Security consideration:
  Traffic between LB and backends is plain HTTP
  Must be within private network — never over public internet
  Optional: re-encrypt with internal certs (SSL bridging)
  mTLS between LB and backends for maximum security

ShopSphere:
  API Gateway handles SSL termination
  Internal service-to-service traffic uses HTTP within the cluster
  Production: Istio service mesh re-encrypts with mTLS internally
```

---

## Connection Draining — Zero Downtime Deployments

When removing a server from rotation (deployment, scaling down), you must handle in-flight requests:

```
Without connection draining:
  LB removes Server A from rotation immediately
  50 active requests on Server A are dropped
  Clients see connection reset errors

With connection draining:
  LB marks Server A as "draining"
  No NEW requests routed to Server A
  Existing 50 requests allowed to complete
  After all requests finish (or timeout) → Server A removed
  Zero requests dropped
  
Spring Boot with Kubernetes:
  preStop hook → sleep 10s (gives LB time to stop routing)
  gracefulShutdownTimeout: 30s (in-flight requests get 30s to complete)
```

---

## Real-World Example — ShopSphere Load Balancer Architecture

```
Internet
    │
    ▼
AWS Application Load Balancer (L7)
    │  SSL termination
    │  Path-based routing
    │  Health checks every 10s
    │
    ├──► /api/** ──────────────► API Gateway pods (3 replicas)
    │                               │  Kubernetes Service
    │                               ├── gateway-pod-1
    │                               ├── gateway-pod-2
    │                               └── gateway-pod-3
    │
    └──► /ws/**  ──────────────► WebSocket Gateway (sticky sessions)

Inside Kubernetes — Service as internal L4 Load Balancer:
  Each microservice has a Kubernetes Service (ClusterIP)
  kube-proxy implements L4 load balancing via iptables/IPVS
  
  order-service Service → round-robin across:
    ├── order-pod-1
    ├── order-pod-2
    └── order-pod-3
```

```yaml
# AWS ALB annotations for ShopSphere
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shopsphere-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "10"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
spec:
  rules:
    - host: api.shopsphere.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8080
```

---

## Interview Q&A

**Q: What is the difference between an L4 and L7 load balancer?**
An L4 load balancer operates at the TCP/UDP transport layer — it sees only source and destination IP addresses and ports, routing raw bytes without understanding the content. It is extremely fast with minimal overhead. An L7 load balancer operates at the HTTP application layer — it can read URLs, headers, cookies, and request bodies, enabling content-based routing, SSL termination, sticky sessions by cookie, and application-aware health checks. Use L4 for raw TCP services and maximum throughput. Use L7 for HTTP microservices where you need intelligent routing and SSL termination.

**Q: What is the difference between round robin and least connections algorithms?**
Round robin distributes requests evenly in rotation regardless of server state — each server gets equal share sequentially. It is simple and works well when requests have similar processing times. Least connections routes each new request to the server with the fewest active connections, accounting for actual current load. It is better when request duration varies significantly — a server processing one expensive long-running upload should not receive new requests while ten other servers handle only short-lived API calls. Weighted variants of both accommodate servers with different capacities.

**Q: What is connection draining and why does it matter?**
Connection draining is the process of gracefully removing a server from rotation without dropping in-flight requests. When a server is marked for removal — during a deployment, scale-down, or failure — the load balancer stops routing new requests to it but allows existing connections to complete naturally up to a timeout. Without draining, removing a server mid-request causes clients to see connection reset errors. With draining, ongoing requests finish successfully and users experience zero disruption. In Kubernetes this is implemented via preStop hooks and graceful shutdown configuration.

**Q: What is the difference between a liveness probe and a readiness probe?**
A liveness probe answers "is the container alive?" — a failure means the container is deadlocked or crashed and should be restarted. A readiness probe answers "is the container ready to receive traffic?" — a failure removes the pod from the load balancer rotation but does not restart it. The distinction is critical for startup — a service might be alive and healthy but still warming up its caches, running database migrations, or waiting for downstream dependencies. During that window it should not receive traffic but should not be killed and restarted either. Readiness probe failure handles this case cleanly.

**Q: How does a load balancer detect that a backend server is unhealthy?**
Active health checks are the reliable approach — the load balancer proactively sends requests to a dedicated health endpoint on each backend at a configured interval, typically every 5 to 10 seconds. In Spring Boot this is the Actuator health endpoint. The server is marked unhealthy after N consecutive failures and removed from rotation immediately, before real requests are affected. It is marked healthy again after M consecutive successes. Passive health checks observe real traffic and mark servers unhealthy after seeing error responses or timeouts, but real requests fail during the detection window — less desirable.

---

Say **"next"** when ready for Topic 2 — Forward Proxy vs Reverse Proxy.
