# Stage 6 — Reliability & Availability
## Topic 10 : Health Checks

---

### Why Health Checks Exist

```
Without health checks:

  Service instance starts up
  Spring Boot is loading beans...
  DB connection pool initializing...
  Kafka consumer connecting...
  → Takes 30 seconds to be ready

  Load balancer doesn't know any of this.
  Load balancer sees: port 8080 is open → send traffic!

  Requests arrive during startup →
  DB pool not ready → NullPointerException →
  Users get 500 errors for 30 seconds on every deploy 💀

  OR:

  Service is running but internally deadlocked.
  Port 8080 still responds to TCP ping.
  Load balancer sees: port open → healthy!
  But every request returns 500 because service is broken.
  Load balancer never removes it. Users keep hitting it. 💀
```

> **Health checks are how infrastructure knows the difference between "port is open" and "service is actually ready to serve traffic correctly."**

---

### The Three Probe Types — Know All Three

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  LIVENESS PROBE                                         │
│  "Is this process alive or should it be restarted?"     │
│                                                         │
│  Fails → Kubernetes KILLS and RESTARTS the container    │
│  Use for: deadlock detection, infinite loop detection   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  READINESS PROBE                                        │
│  "Is this instance ready to receive traffic?"           │
│                                                         │
│  Fails → Kubernetes REMOVES from load balancer pool     │
│          (container keeps running, just no traffic)     │
│  Use for: startup warmup, dependency checks,            │
│           graceful degradation                          │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  STARTUP PROBE                                          │
│  "Has the application finished initializing?"           │
│                                                         │
│  While failing → liveness probe is DISABLED             │
│  Once passes  → liveness takes over                     │
│  Use for: slow-starting apps (Spring Boot, JVM warmup)  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

### Liveness vs Readiness — The Critical Distinction

This is the most misunderstood concept. One wrong config = cascading restarts.

```
LIVENESS — answers: "Should this process continue existing?"

  Fails when:
    ├── JVM deadlocked (all threads stuck)
    ├── Infinite loop consuming all CPU
    ├── Application entered unrecoverable state
    └── OOM killer hit, process corrupted

  Does NOT fail when:
    ├── Database is temporarily down ← WRONG to fail liveness here
    ├── Redis is unreachable         ← WRONG to fail liveness here
    └── Downstream service is slow  ← WRONG to fail liveness here

  Why? If liveness fails → container restarts.
  DB is down → all instances restart → they all come back up →
  DB still down → all restart again → restart loop 💀
  You just made a DB outage into a full service outage.


READINESS — answers: "Should traffic come to this instance?"

  Fails when:
    ├── DB connection pool exhausted
    ├── Startup not complete yet
    ├── Cache warmup in progress
    ├── Downstream critical dependency unhealthy
    └── Instance under too much load (circuit breakers open)

  Does NOT restart the container.
  Just stops sending traffic here.
  Other healthy instances absorb the load. ✅
```

```
The rule:
  Liveness  = "is the PROCESS healthy?"   → restart if not
  Readiness = "is the SERVICE healthy?"   → remove from LB if not
```

---

### Spring Boot Actuator Health — The Foundation

Spring Boot Actuator provides health endpoints out of the box.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      show-details: always          # show component breakdown
      show-components: always
      probes:
        enabled: true               # enables /health/liveness
                                    #         /health/readiness

  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true

    # Component-level health config
    db:
      enabled: true                 # checks DB connection
    redis:
      enabled: true                 # checks Redis ping
    kafka:
      enabled: true                 # checks Kafka broker
    diskspace:
      enabled: true
      threshold: 10GB               # fail if < 10GB free
```

**Endpoints exposed:**
```
GET /actuator/health           → overall health (all components)
GET /actuator/health/liveness  → liveness state only
GET /actuator/health/readiness → readiness state only
```

**Sample responses:**
```json
// GET /actuator/health/liveness
// Used by Kubernetes liveness probe
{
  "status": "UP"
}

// GET /actuator/health/readiness
// Used by Kubernetes readiness probe
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "PostgreSQL", "validationQuery": "isValid()" }
    },
    "redis": {
      "status": "UP",
      "details": { "version": "7.2.4" }
    },
    "kafka": {
      "status": "UP",
      "details": { "bootstrapServers": "kafka:9092" }
    },
    "readinessState": {
      "status": "UP"
    }
  }
}

// When DB is down:
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": { "error": "Connection refused" }
    },
    "redis": { "status": "UP" },
    "readinessState": { "status": "UP" }
  }
}
```

---

### Custom Health Indicators — ShopSphere Patterns

Built-in health checks aren't always enough. You need **domain-aware** checks.

```java
// Custom health indicator for Elasticsearch (Search Service)
@Component
public class ElasticsearchHealthIndicator implements HealthIndicator {

    private final ElasticsearchClient esClient;

    @Override
    public Health health() {
        try {
            // Ping Elasticsearch cluster
            HealthResponse response = esClient.cluster().health(
                h -> h.timeout(t -> t.time("3s"))
            );

            ClusterHealthStatus clusterStatus = response.status();

            if (clusterStatus == ClusterHealthStatus.Red) {
                // RED = data loss or shards unassigned → not ready
                return Health.down()
                    .withDetail("cluster_status", "RED")
                    .withDetail("reason", "Primary shards unassigned")
                    .build();
            }

            if (clusterStatus == ClusterHealthStatus.Yellow) {
                // YELLOW = replicas unassigned → degraded but functional
                return Health.status("DEGRADED")
                    .withDetail("cluster_status", "YELLOW")
                    .withDetail("reason", "Replica shards unassigned")
                    .withDetail("impact", "Reduced redundancy")
                    .build();
            }

            // GREEN = fully healthy
            return Health.up()
                .withDetail("cluster_status", "GREEN")
                .withDetail("node_count", response.numberOfNodes())
                .build();

        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .withDetail("reason", "Cannot reach Elasticsearch")
                .build();
        }
    }
}
```

```java
// Custom health indicator — checks critical Kafka topics exist
@Component
public class KafkaTopicsHealthIndicator implements HealthIndicator {

    private final AdminClient kafkaAdmin;

    private static final List<String> REQUIRED_TOPICS = List.of(
        "order.created",
        "order.confirmed",
        "payment.processed",
        "inventory.updated"
    );

    @Override
    public Health health() {
        try {
            Set<String> existingTopics = kafkaAdmin
                .listTopics()
                .names()
                .get(3, TimeUnit.SECONDS);    // 3s timeout

            List<String> missingTopics = REQUIRED_TOPICS.stream()
                .filter(t -> !existingTopics.contains(t))
                .toList();

            if (!missingTopics.isEmpty()) {
                return Health.down()
                    .withDetail("missing_topics", missingTopics)
                    .withDetail("reason", "Required topics not found")
                    .build();
            }

            return Health.up()
                .withDetail("topics_verified", REQUIRED_TOPICS)
                .build();

        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

```java
// Manual readiness control — useful during graceful shutdown
@Component
public class ShopSphereReadinessController {

    private final ApplicationEventPublisher eventPublisher;

    // Called during graceful shutdown — stop taking traffic
    // before stopping the process
    public void markNotReady() {
        eventPublisher.publishEvent(
            new AvailabilityChangeEvent<>(this,
                ReadinessState.REFUSING_TRAFFIC)
        );
    }

    // Called after warmup completes — ready for traffic
    public void markReady() {
        eventPublisher.publishEvent(
            new AvailabilityChangeEvent<>(this,
                ReadinessState.ACCEPTING_TRAFFIC)
        );
    }
}
```

---

### Startup Probe — For JVM Services

JVM services (especially Spring Boot) are **slow to start.**
Without a startup probe, liveness kicks in during startup and kills the container.

```
Without startup probe:
  t=0s    Container starts, JVM loading
  t=5s    Liveness probe fires → /health/liveness
  t=5s    Spring still loading → no response → FAIL
  t=5s    Kubernetes: "container is dead, restart it"
  t=5s    Container restarted → JVM loading again
  t=10s   Liveness fires again → still loading → FAIL
  t=10s   Restart again → CrashLoopBackOff 💀

With startup probe:
  t=0s    Container starts
  t=0–40s Startup probe runs every 5s, allows up to 8 failures
           Liveness probe DISABLED during this window
  t=32s   Spring Boot fully loaded → startup probe passes ✅
  t=32s   Startup probe done → liveness now takes over
  t=32s+  Liveness runs normally
```

---

### Kubernetes Probe Configuration — ShopSphere

```yaml
# order-service deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: order-service
          image: shopsphere/order-service:latest
          ports:
            - containerPort: 8080

          # ── STARTUP PROBE ──────────────────────────────────
          # JVM + Spring Boot can take 30-45s to start
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10   # wait 10s before first check
            periodSeconds: 5          # check every 5s
            failureThreshold: 12      # allow 12 failures
                                      # = 10 + (12×5) = 70s max startup
            successThreshold: 1
            timeoutSeconds: 3

          # ── LIVENESS PROBE ─────────────────────────────────
          # Only checks if JVM/process is alive
          # Does NOT check DB, Redis, Kafka
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 0    # startup probe guards this
            periodSeconds: 10         # check every 10s
            failureThreshold: 3       # 3 failures = restart
                                      # = 30s to detect deadlock
            successThreshold: 1
            timeoutSeconds: 5

          # ── READINESS PROBE ────────────────────────────────
          # Checks DB, Redis, Kafka — all dependencies
          # More aggressive — remove from LB faster
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5          # check every 5s
            failureThreshold: 3       # 3 failures = remove from LB
                                      # = 15s to pull out of rotation
            successThreshold: 2       # must pass twice to re-add
                                      # prevents flapping
            timeoutSeconds: 3

          # Resource limits (affects scheduling + OOM kill)
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

---

### Probe Failure Scenarios — What Actually Happens

```
Scenario 1: Rolling Deploy

  Pod A (old): Readiness = UP, serving traffic
  Pod B (new): Starting up → Readiness = DOWN

  Kubernetes: Don't send traffic to B yet
  Traffic 100% → Pod A

  Pod B: Startup complete → Readiness = UP
  Kubernetes: Add B to load balancer
  Traffic 50/50 → A and B

  Pod A: Terminated (old version)
  Traffic 100% → Pod B ✅

  Zero dropped requests during deploy


Scenario 2: DB goes down

  Pod A: Readiness probe → DB DOWN → probe FAILS
  Pod B: Readiness probe → DB DOWN → probe FAILS
  Pod C: Readiness probe → DB DOWN → probe FAILS

  Kubernetes: Remove ALL pods from load balancer
  Load balancer returns 503 to clients

  BUT: Liveness probe → /health/liveness (no DB check)
  Kubernetes: Liveness still UP → no restarts ✅
  Pods kept alive, waiting for DB recovery

  DB recovers → Readiness passes → traffic resumes ✅

  If liveness also checked DB → mass restart loop 💀


Scenario 3: JVM deadlock

  Pod A: Liveness probe → /actuator/health/liveness → no response
  Timeout fires (5s) → counted as failure
  3 failures → Kubernetes KILLS and RESTARTS container ✅

  Deadlock cleared by restart
  Auto-healed without human intervention ✅
```

---

### Graceful Shutdown — The Underrated Health Check Feature

```
Without graceful shutdown:
  Kubernetes sends SIGTERM to pod
  Pod immediately dies
  In-flight requests: 💀 connection reset
  Users see errors

With graceful shutdown:

  1. Kubernetes sends SIGTERM
  2. Spring Boot: set Readiness = REFUSING_TRAFFIC
                  (removes from load balancer)
  3. No new requests arrive
  4. Existing in-flight requests complete
  5. After completion (or timeout): process exits cleanly
```

```yaml
# application.yml — graceful shutdown
server:
  shutdown: graceful          # Spring Boot 2.3+

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s   # wait up to 30s for in-flight

# Kubernetes pod terminationGracePeriodSeconds must be
# GREATER than Spring's timeout
# terminationGracePeriodSeconds: 60  ← in deployment yaml
```

```java
// Spring automatically sets readiness to REFUSING_TRAFFIC
// when SIGTERM received — no code needed with:
// server.shutdown=graceful

// But you can also do it manually:
@EventListener(ContextClosedEvent.class)
public void onShutdown() {
    log.info("Shutdown initiated — marking not ready");
    readinessController.markNotReady();
    // Spring then waits for in-flight requests before stopping
}
```

---

### Health Check Anti-Patterns

```
❌ Anti-pattern 1: Checking external dependencies in liveness
   livenessProbe → /health (includes DB check)
   DB goes down → liveness fails → all pods restart →
   DB still down → restart loop → CrashLoopBackOff

   Fix: Liveness checks ONLY process health
        Readiness checks dependencies

❌ Anti-pattern 2: No successThreshold on readiness
   DB flapping (up/down every 5s)
   readiness oscillates → pod added/removed rapidly
   Load balancer thrashing → inconsistent routing

   Fix: successThreshold: 2 (must pass twice to re-add)
        Adds hysteresis — prevents flapping

❌ Anti-pattern 3: Startup timeout too tight
   startupProbe failureThreshold: 6, period: 5s = 30s max
   JVM + Spring Boot = 35s cold start
   Pod killed at 30s → restart → killed again → loop

   Fix: Be generous: failureThreshold: 12, period: 5s = 60s
        Better to wait longer than restart-loop forever

❌ Anti-pattern 4: Health endpoint itself is expensive
   /health runs DB query on every probe call
   Probe fires every 5s across 10 pods = 2 queries/sec
   Under load: health check competes with real traffic

   Fix: Use Actuator's built-in connection validation (isValid())
        Not SELECT * FROM orders
        Cache health state, refresh every 10s internally
```

---

### ShopSphere Health Check Map 🛒

```
┌──────────────────────────────────────────────────────────┐
│              ShopSphere Health Check Strategy            │
│                                                          │
│  Service          Liveness         Readiness             │
│  ──────────────────────────────────────────────────────  │
│  Order Service    process UP        DB ✓ Redis ✓ Kafka ✓ │
│  Payment Service  process UP        DB ✓ Redis ✓         │
│  Product Service  process UP        DB ✓ Redis ✓         │
│  Search Service   process UP        ES cluster ✓         │
│  Notification     process UP        Kafka ✓ SMTP ping ✓  │
│  API Gateway      process UP        routes reachable ✓   │
│                                                          │
│  Startup probe:   all services — 70s window (JVM)        │
│  Liveness:        /actuator/health/liveness — 10s period │
│  Readiness:       /actuator/health/readiness — 5s period │
│                                                          │
│  Graceful shutdown: 30s drain window on all services     │
│  terminationGracePeriodSeconds: 60 (Kubernetes)          │
│                                                          │
│  Custom indicators:                                      │
│  Search  → ElasticsearchHealthIndicator (RED/YELLOW/GREEN│
│  Order   → KafkaTopicsHealthIndicator (required topics)  │
│  Payment → CircuitBreakerHealthIndicator (CB states)     │
└──────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What is the difference between liveness and readiness probes?**
> Liveness answers "is the process alive?" — failure triggers a container restart. Readiness answers "is the instance ready to receive traffic?" — failure removes it from the load balancer without restarting. The critical rule is never check external dependencies like DB or Redis in liveness. If DB goes down and liveness fails, all pods restart in a loop — DB is still down, pods keep restarting. Readiness handles dependency failures correctly by pulling the pod from rotation while keeping it alive.

**Q2. Why do you need a startup probe separate from liveness?**
> JVM services take 30–60 seconds to start. Without a startup probe, liveness kicks in during startup, sees no response, and kills the container — which then restarts and gets killed again in a crash loop. Startup probe disables liveness until the application passes its first health check, giving slow-starting services the time they need to fully initialize before the liveness watchdog takes over.

**Q3. What happens to in-flight requests during a rolling deploy without graceful shutdown?**
> When Kubernetes sends SIGTERM, the pod dies immediately. Any requests currently being processed get a connection reset — users see errors. With graceful shutdown (`server.shutdown=graceful`), Spring Boot marks readiness as REFUSING_TRAFFIC first, stopping new requests from routing to this pod, then waits for in-flight requests to complete before the process exits. Zero dropped requests during deploy.

**Q4. A pod's readiness probe is flapping — going up and down every few seconds. What's happening and how do you fix it?**
> A dependency (DB, Redis) is intermittently failing health checks — possibly high latency, brief timeouts, or connection pool exhaustion. The pod gets rapidly added and removed from the load balancer causing unstable routing. Fix: set `successThreshold: 2` so a pod must pass two consecutive probes before re-entering rotation. This adds hysteresis — prevents oscillation while still pulling pods out quickly on sustained failure.

**Q5. Should health check endpoints be secured?**
> Liveness and readiness endpoints should be accessible without authentication from within the cluster — Kubernetes probes don't send credentials. However, the full `/actuator/health` endpoint with details (DB credentials in error messages, internal topology) should be secured externally. Best practice: expose probes on a separate management port (e.g., 8081) not exposed outside the cluster, while the main API port (8080) is externally accessible but has actuator endpoints filtered or secured.

---

### One-Line Summary

> **Liveness = restart the broken process, Readiness = remove it from traffic — never mix them. Startup probe gives JVM time to wake up, graceful shutdown gives in-flight requests time to finish, and custom health indicators make your checks domain-aware instead of just port-pinging.**

