# Stage 6 — Reliability & Availability
## Topic 9 : SLA / SLO / SLI

---

### The Three-Layer Reliability Contract

Most engineers confuse these three. They form a strict hierarchy:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   SLI  →  the raw measurement                       │
│   SLO  →  the internal target on that measurement   │
│   SLA  →  the external contract with consequences   │
│                                                     │
│   SLI is what you MEASURE                           │
│   SLO is what you AIM FOR                           │
│   SLA is what you PROMISE and PAY for if broken     │
│                                                     │
└─────────────────────────────────────────────────────┘

Real world analogy:

  SLI: Your car's speedometer reading       (raw metric)
  SLO: "I'll drive under 120 km/h"          (internal target)
  SLA: "I'll deliver your package by 6 PM"  (external contract)
       (if I don't → you get a refund)
```

---

### SLI — Service Level Indicator

> **SLI = a carefully defined quantitative measure of some aspect of the level of service being provided.**

It is a **ratio** — always expressed as:

```
SLI = (good events) / (total events) × 100

Where "good" is precisely defined per SLI type.
```

**The four canonical SLI categories (Google SRE):**

```
1. AVAILABILITY SLI
   Good event  = request returned successful response (2xx, 3xx)
   Bad event   = request returned error (5xx) or timed out
   
   SLI = (non-5xx responses) / (total requests) × 100

2. LATENCY SLI
   Good event  = request completed within threshold (e.g. 300ms)
   Bad event   = request took longer than threshold
   
   SLI = (requests < 300ms) / (total requests) × 100

3. THROUGHPUT SLI
   Good event  = data processed exceeds minimum rate
   Used for: data pipelines, Kafka consumers, batch jobs

4. CORRECTNESS SLI
   Good event  = response contains correct / expected data
   Used for: ML systems, data pipelines, calculation services
```

**Measuring SLI correctly — percentiles matter:**

```
BAD: "Average latency = 50ms"
  90% of requests: 10ms
  10% of requests: 410ms
  Average: (0.9×10) + (0.1×410) = 9 + 41 = 50ms ← looks fine!
  But 10% of users are miserable

GOOD: "p99 latency < 300ms"
  99th percentile — the worst 1% of requests
  If p99 < 300ms → 99 out of 100 users happy

Percentile ladder:
  p50  (median) → typical user experience
  p90           → most users
  p95           → almost all users
  p99           → 1 in 100 users at worst
  p99.9         → 1 in 1000 users at worst ← used for payments

ShopSphere SLIs:
  Availability SLI = (2xx + 3xx responses) / total × 100
  Latency SLI (checkout) = requests completing < 500ms / total × 100
  Latency SLI (search)   = requests completing < 200ms / total × 100
```

---

### SLO — Service Level Objective

> **SLO = the target value or range for an SLI. Your internal reliability goal.**

```
SLI → SLO

Availability SLI ──► SLO: 99.9% of requests succeed
                          measured over 30-day rolling window

Latency SLI      ──► SLO: 95% of checkout requests < 500ms
                          p99 of checkout requests < 2s

Format:
  "X% of [SLI events] will meet [threshold]
   over [measurement window]"

Examples:
  "99.9% of HTTP requests to /api/orders will return
   non-5xx responses over any 30-day window"

  "95% of search requests will complete in < 200ms
   measured over a 7-day rolling window"
```

**SLO should be STRICTER than SLA:**

```
  SLA (external promise): 99.9% availability
  SLO (internal target):  99.95% availability

  Why the gap?
  SLO gives you a WARNING before you breach your SLA.
  If SLO is breached → alert fires → fix it.
  If you wait for SLA breach → customer already affected
                             → penalty already incurred.

  ├── 100% ──────────────────────────────────────────
  │
  ├── 99.95% ─────────── SLO target (internal)
  │                ← buffer zone: fix things here
  ├── 99.9% ──────────── SLA threshold (external)
  │                ← if you cross this → credits owed
  ├── 99.5% ──────────── "we have a serious problem"
  │
  └── 0% ────────────────────────────────────────────
```

---

### Error Budget — The SLO's Most Powerful Concept

> **Error budget = 100% - SLO. The amount of unreliability you're ALLOWED to have.**

```
SLO = 99.9% over 30 days

30-day window = 30 × 24 × 60 = 43,200 minutes

Error budget = 0.1% of 43,200 = 43.2 minutes of downtime allowed

That's your budget. You can spend it on:
  ├── Planned deploys that cause brief disruption
  ├── Experiments and feature flags
  ├── Risky but necessary infrastructure changes
  └── Unplanned incidents (bugs, cascades)

Error budget tracking:

  Week 1: deploy caused 5 min downtime   → 38.2 min remaining
  Week 2: incident caused 10 min         → 28.2 min remaining
  Week 3: everything clean               → 28.2 min remaining
  Week 4: big deploy, 15 min downtime    → 13.2 min remaining

  Budget still positive → SLO met ✅
```

**Error budget policy — what changes when budget is low:**

```
Budget remaining > 50%:
  ✅ New feature deploys allowed
  ✅ Experiments in production allowed
  ✅ Infrastructure changes allowed

Budget remaining 10–50%:
  ⚠️  Risky deploys need extra review
  ⚠️  Chaos engineering paused
  ✅  Critical bug fixes still proceed

Budget remaining < 10%:
  ❌  Feature deploys FROZEN
  ❌  Only reliability fixes allowed
  📢  Engineering leadership notified
  🔍  Post-mortem on recent incidents mandatory

Budget exhausted (0%):
  🚨  All non-critical work stopped
  🚨  SRE and engineering on reliability sprint
  📋  Formal incident review with management
```

---

### SLA — Service Level Agreement

> **SLA = a formal contract between a service provider and customer, defining the minimum service level and consequences of breaching it.**

```
SLA components:

  1. SCOPE          → which services / endpoints are covered
  2. SLI DEFINITION → exactly how availability is measured
  3. THRESHOLD      → the minimum acceptable value
  4. MEASUREMENT    → time window, data source, exclusions
  5. REMEDIES       → what happens when breached
                      (service credits, refunds, exit clauses)
  6. EXCLUSIONS     → what doesn't count against SLA
                      (scheduled maintenance, force majeure,
                       customer-caused outages)
```

**Real SLA examples:**

```
AWS EC2 SLA:
  "Monthly Uptime Percentage < 99.99% → 10% service credit
   Monthly Uptime Percentage < 99.0%  → 30% service credit"

  Measurement: 5-minute intervals, majority of regions
  Exclusions: Scheduled maintenance, customer actions

Google Cloud SLA:
  GCE: 99.99% monthly uptime
  GKE: 99.5% (zonal) / 99.95% (regional)
  Credits: 10% → 25% → 50% based on severity

Stripe Payment API:
  99.99% uptime for payment processing
  Credits for downtime exceeding threshold
```

**ShopSphere SLA (if B2B product):**
```
  Payment API:   99.99% monthly uptime
                 p99 latency < 1000ms
                 Breach → 10% monthly credit

  Order API:     99.9% monthly uptime
                 p95 latency < 500ms
                 Breach → 5% monthly credit

  Search API:    99.5% monthly uptime
                 No latency SLA
                 Breach → 2% monthly credit

  Excluded:
    - Maintenance windows (announced 48hr prior, max 4hr/month)
    - Incidents caused by third-party payment processors
    - Force majeure events
```

---

### Measuring SLIs in Production — The Stack

```
Request arrives
     │
     ▼
  Service processes
     │
     ▼
  Metrics emitted (Micrometer → Prometheus)
     │
     ├── http_requests_total{status="200"} counter++
     ├── http_requests_total{status="500"} counter++
     └── http_request_duration_seconds histogram observation

  Prometheus scrapes every 15s

  Grafana queries:
    Availability SLI:
      sum(rate(http_requests_total{status!~"5.."}[5m]))
      /
      sum(rate(http_requests_total[5m]))

    Latency SLI (p99):
      histogram_quantile(0.99,
        sum(rate(http_request_duration_seconds_bucket[5m]))
        by (le))
```

**Spring Boot + Micrometer (ShopSphere):**
```java
@RestController
public class OrderController {

    private final MeterRegistry registry;

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody CreateOrderRequest request) {

        Timer.Sample sample = Timer.start(registry);
        String status = "success";

        try {
            OrderResponse response = orderService.create(request);
            return ResponseEntity.ok(response);

        } catch (BusinessException e) {
            status = "client_error";    // 4xx — doesn't hurt SLI
            throw e;

        } catch (Exception e) {
            status = "server_error";    // 5xx — hurts SLI
            throw e;

        } finally {
            // Record latency with tags
            sample.stop(Timer.builder("order.request.duration")
                .tag("status", status)
                .tag("endpoint", "create_order")
                .register(registry));

            // Increment request counter
            registry.counter("order.requests.total",
                "status", status).increment();
        }
    }
}
```

---

### SLO Alerting — Burn Rate

> **Don't alert on instantaneous SLI dips. Alert when error budget is burning too fast.**

```
Concept: Burn Rate

  SLO: 99.9% over 30 days → budget = 43.2 minutes

  Normal burn rate = 1×
    consuming budget at exactly the rate that
    uses it all in 30 days

  2× burn rate:
    budget exhausted in 15 days — problem

  10× burn rate:
    budget exhausted in 3 days — page someone NOW

  60× burn rate:
    budget exhausted in 12 hours — wake someone up
```

**Multi-window alerting (Google SRE recommendation):**

```yaml
# Prometheus alerting rules

# CRITICAL: Fast burn — 2% budget in 1 hour
# (burn rate 14.4×)
- alert: SLOFastBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
      /
      sum(rate(http_requests_total[1h]))
    ) > 14.4 * 0.001   # 14.4x the allowed error rate
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "SLO error budget burning too fast"
    description: "At current rate budget exhausted in < 1hr"

# WARNING: Slow burn — 5% budget in 6 hours
# (burn rate 6×)
- alert: SLOSlowBurn
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[6h]))
      /
      sum(rate(http_requests_total[6h]))
    ) > 6 * 0.001
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "SLO error budget elevated burn"
```

---

### SLO Dashboard — What ShopSphere Tracks

```
┌─────────────────────────────────────────────────────────┐
│            ShopSphere SLO Dashboard (30-day)            │
│                                                         │
│  Service          SLI       SLO    Current  Budget Left │
│  ─────────────────────────────────────────────────────  │
│  Payment API      Avail    99.99%  99.996%  ████░ 68%   │
│  Payment API      p99 lat  <1000ms  420ms   ██████ 82%  │
│                                                         │
│  Order API        Avail    99.9%   99.95%   █████░ 71%  │
│  Order API        p95 lat  <500ms  180ms    ████░░ 60%  │
│                                                         │
│  Search API       Avail    99.5%   99.8%    ████████92% │
│  Search API       p95 lat  <200ms  140ms    ███████ 88% │
│                                                         │
│  ⚠️  Order API latency budget moderate — review deploys  │
└─────────────────────────────────────────────────────────┘
```

---

### Interview Questions 🎯

**Q1. What is the difference between SLI, SLO, and SLA?**
> SLI is the raw measurement — a ratio of good events to total events (e.g., non-5xx requests / total requests). SLO is the internal target on that SLI — "99.9% of requests succeed over 30 days." SLA is the external contract with a customer — "we guarantee 99.9% uptime or issue service credits." SLO should always be stricter than SLA so you catch problems before customers do.

**Q2. What is an error budget and how does it influence engineering decisions?**
> Error budget is 100% minus the SLO — the allowed unreliability. For a 99.9% SLO over 30 days, that's 43.2 minutes. When budget is healthy, teams can take risks — deploy often, run experiments, make infrastructure changes. When budget is low, risky deploys freeze and reliability work takes priority. It turns "how reliable should we be?" from a philosophical debate into a data-driven engineering policy.

**Q3. Why use burn rate alerts instead of threshold alerts for SLOs?**
> A threshold alert on instantaneous error rate fires on every small blip — noisy, leads to alert fatigue. Burn rate tells you how fast you're consuming your error budget. A 14× burn rate means budget exhausted in 1 hour — worth waking someone up. A 1× burn rate means you're on track — no alert needed. Multi-window burn rate (1h + 6h) catches both fast burns and slow leaks.

**Q4. What should be excluded from SLA measurement?**
> Announced maintenance windows (typically capped at X hours/month), incidents caused by the customer's own actions, force majeure events (natural disasters, ISP-wide outages), and third-party dependency failures outside your control. The SLA document must explicitly list these or customers will count every minute of downtime against you.

**Q5. A team is consistently hitting 99.95% availability against a 99.9% SLO. Should they relax the SLO?**
> No — that's the error budget working as intended. The gap between actual (99.95%) and SLO (99.9%) is the buffer that protects against bad weeks. If you tighten the SLO to 99.95%, you eliminate that buffer and any incident immediately threatens SLA breach. Instead, use the healthy budget to invest in riskier reliability improvements that could push availability to 99.99% over time.

---

### One-Line Summary

> **SLI measures it, SLO targets it, SLA contracts it — error budget is the SLO's gift to engineering: a quantified amount of allowed unreliability that tells teams exactly when to ship features versus fix reliability.**

---

Ready for **Topic 10: Health Checks** — liveness vs readiness probes, how load balancers and Kubernetes use them, startup probes, and the exact patterns ShopSphere implements across all services?

Type **next** 🚀
