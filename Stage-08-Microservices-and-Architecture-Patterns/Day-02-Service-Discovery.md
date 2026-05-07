## Topic 2 — Service Discovery

### The Problem First

In a monolith, components talk to each other in-memory. In microservices, services communicate over the network. But here's the issue:

```
Order Service needs to call Product Service.
What IP and port is Product Service running on?

In production with Kubernetes:
  product-service pod 1 → 10.0.1.45:8080
  product-service pod 2 → 10.0.1.67:8080
  product-service pod 3 → 10.0.1.89:8080

Pods die, restart, get rescheduled → IPs change constantly.
You CANNOT hardcode IPs.
```

This is the **dynamic service location problem**. Service Discovery solves it.

---

### What is Service Discovery?

A mechanism where services **register themselves** and **others look them up by name**, not by IP.

```
Instead of:  http://10.0.1.45:8080/api/products
You call:    http://product-service/api/products

The discovery layer resolves the name → current healthy IP at runtime.
```

---

### Two Models

#### Model 1 — Client-Side Discovery

The **client** itself queries the service registry, gets the list of healthy instances, picks one (load balances), and calls it directly.

```
┌──────────────┐     1. "Where is product-service?"    ┌──────────────────┐
│ Order        │ ──────────────────────────────────────▶│ Service Registry │
│ Service      │ ◀──────────────────────────────────────│ (Eureka/Consul)  │
│ (client)     │     2. "Here are 3 healthy instances"  └──────────────────┘
└──────┬───────┘
       │  3. Client picks one (round-robin, least-conn)
       ▼
┌──────────────┐
│ Product      │
│ Service      │
│ (chosen pod) │
└──────────────┘
```

**Used by:** Netflix Eureka + Ribbon (old Spring Cloud stack)

**Drawback:** Every client (every service) needs discovery logic built in. Couples your service code to the registry client library.

---

#### Model 2 — Server-Side Discovery

The **client calls a load balancer or proxy**. The proxy does the registry lookup and forwards the request. Client knows nothing about discovery.

```
┌──────────────┐   1. Calls "product-service"   ┌─────────────────┐
│ Order        │ ──────────────────────────────▶ │  Load Balancer  │
│ Service      │                                  │  or API Gateway │
└──────────────┘                                  └────────┬────────┘
                                                           │  2. Looks up registry
                                                           ▼
                                                  ┌─────────────────┐
                                                  │ Service Registry│
                                                  └────────┬────────┘
                                                           │  3. Routes to healthy pod
                                                           ▼
                                                  ┌─────────────────┐
                                                  │ Product Service │
                                                  └─────────────────┘
```

**Used by:** AWS ALB, Kubernetes (kube-proxy + Services), Istio

---

### The Three Major Tools

#### 1. Eureka (Netflix / Spring Cloud)

The classic Java microservices registry. Each service has a Eureka client that:
- On startup → **registers** itself (name, IP, port, health URL)
- Every 30s → sends **heartbeat** to say "I'm alive"
- If heartbeat stops → Eureka marks it **DOWN** after 90s and removes it

```yaml
# In each service's application.yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka/
  instance:
    prefer-ip-address: true
```

```
Eureka Registry State:
┌─────────────────────────────────────────────┐
│ Application     │ Status  │ IP            │  │
├─────────────────────────────────────────────┤
│ ORDER-SERVICE   │ UP      │ 10.0.1.12:8080│  │
│ PRODUCT-SERVICE │ UP      │ 10.0.1.45:8080│  │
│ PRODUCT-SERVICE │ UP      │ 10.0.1.67:8080│  │
│ USER-SERVICE    │ DOWN    │ 10.0.1.99:8080│  │
└─────────────────────────────────────────────┘
```

Order Service calls `http://PRODUCT-SERVICE/api/products` → Spring Cloud LoadBalancer picks one of the two UP instances.

**Problem with Eureka:** It's eventually consistent. If a pod crashes, Eureka takes up to 90s to notice. During that window, it still sends traffic to the dead pod.

---

#### 2. Consul (HashiCorp)

More production-grade than Eureka. Built for multi-datacenter, supports health checks, KV store, and service mesh.

```
Consul agent runs on EVERY node (sidecar-like).
Each service registers with its local Consul agent.
Agents gossip state to each other (Serf protocol).

consul register:
  {
    "name": "product-service",
    "address": "10.0.1.45",
    "port": 8080,
    "check": {
      "http": "http://10.0.1.45:8080/actuator/health",
      "interval": "10s"
    }
  }
```

Consul actively **HTTP health-checks** your service every 10s. If `/actuator/health` returns non-200, it marks the service critical and removes it from results immediately — much faster than Eureka's heartbeat model.

Also supports **DNS-based discovery**: `product-service.service.consul` resolves to healthy IPs automatically.

---

#### 3. Kubernetes DNS (Most Relevant for ShopSphere)

In Kubernetes, every **Service** object gets a stable DNS name. No external registry needed — Kubernetes IS the registry.

```
You deploy product-service as a K8s Service:

apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: shopsphere
spec:
  selector:
    app: product-service
  ports:
    - port: 8080

→ Kubernetes creates DNS entry:
  product-service.shopsphere.svc.cluster.local
  (or just "product-service" within the same namespace)
```

When Order Service calls `http://product-service:8080/api/products`:

```
Order pod
  │
  ├─ DNS query: "product-service" 
  │              ↓
  │         kube-dns / CoreDNS
  │              ↓
  │         Returns ClusterIP: 10.96.45.22
  │              ↓
  ├─ kube-proxy intercepts traffic to 10.96.45.22
  │              ↓
  └─ Routes to one of the healthy product-service pods via iptables/IPVS
```

**The magic:** When a pod dies and a new one starts with a new IP, the **Service's selector** automatically includes the new pod. DNS always returns the Service's stable ClusterIP. Zero configuration changes needed.

```
ShopSphere in K8s:
  order-service calls → http://product-service:8080  ✅
  order-service calls → http://user-service:8080     ✅
  gateway calls       → http://order-service:8080    ✅

No Eureka needed. K8s DNS IS your service discovery.
```

---

### Self-Registration vs Third-Party Registration

```
Self-Registration:
  Service itself calls registry on startup/shutdown.
  Simple, but service must include registry client code.
  (Eureka model)

Third-Party Registration:
  An external watcher (like a K8s controller) detects
  service start/stop and updates the registry.
  Service code stays clean.
  (Kubernetes model — kubelet registers pods)
```

---

### What Happens Without Service Discovery?

```
Without it, you'd hardcode:
  order-service → calls http://192.168.1.45:8080/products

Pod restarts → new IP: 192.168.1.67
→ Order service still calling .45 → Connection refused
→ Manual config update needed → downtime

In K8s with 50 services × 3 replicas = 150 pods
all with dynamic IPs → impossible to manage without discovery
```

---

### ShopSphere Lens

```
Your current setup (Docker Compose):
  Services call each other by container name:
  http://product-service:8080 → Docker DNS resolves it
  This IS service discovery — Docker's built-in DNS

When you move ShopSphere to Kubernetes:
  K8s Service objects take over.
  Spring Cloud Eureka becomes unnecessary.
  Your Feign clients stay the same:
    @FeignClient(name = "product-service")
    → Spring resolves via K8s DNS automatically
      if spring-cloud-starter-kubernetes-client is on classpath
      OR you just set the URL directly to the K8s service name
```

---

### Interview Questions

**Q1. What is service discovery and why is it needed in microservices?**

> In microservices, service instances are dynamic — IPs change as pods restart or scale. Service discovery lets services find each other by **logical name** rather than hardcoded IP. The registry maintains a live map of name → healthy instances.

**Q2. Client-side vs server-side discovery — which is better?**

> Server-side (K8s, proxy-based) is generally preferred in modern stacks because discovery logic is kept **outside service code**, making services simpler and language-agnostic. Client-side gives more control (custom LB logic) but couples every service to the registry library.

**Q3. How does Kubernetes handle service discovery without Eureka?**

> Every K8s Service object gets a stable DNS entry via CoreDNS. Pods call each other by service name. kube-proxy maintains iptables rules that route traffic to healthy pods. When pods scale or restart, the service selector automatically includes new pods — no registry update needed.

**Q4. What happens if the service registry goes down?**

> This is the **registry as SPOF** problem. Eureka handles it with **self-preservation mode** — if it loses heartbeats from too many services at once (network partition), it stops evicting them assuming it's a network issue, not actual failures. Consul uses a **Raft consensus cluster** for high availability. K8s CoreDNS runs as a replicated deployment — if one replica dies, others serve DNS.

---

Ready for **Topic 3 — Sidecar Pattern** when you are.
