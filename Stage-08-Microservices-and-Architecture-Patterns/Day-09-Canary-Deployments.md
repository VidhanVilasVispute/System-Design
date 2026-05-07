## Topic 9 — Canary Deployments

### The Problem With Blue-Green for Every Release

Blue-Green is powerful but binary — you flip 100% of traffic at once:

```
Blue-Green flip:
  Before: 100% → v1
  After:  100% → v2

If v2 has a subtle bug that only appears under real production load:
  - Synthetic smoke tests passed ✅
  - Load testing passed ✅
  - But real user behavior triggers an edge case
  → 100% of users are now hitting the bug
  → You catch it after impact, not before

Some bugs only appear when:
  - Real user data (not test data) flows through
  - Production scale (10k req/s, not 100 in staging)
  - Specific geographic region or device type
  - Combination of feature flags + user preferences
```

You need a way to expose the new version to **real production traffic** — but only a small slice — before committing fully.

---

### What is a Canary Deployment?

Release the new version to a **small percentage of real users first**. Watch metrics. If healthy, gradually expand. If problems appear, roll back before most users are affected.

```
Named after "canary in a coal mine" —
miners sent canaries ahead to detect toxic gas.
If the canary died → danger, don't proceed.
If the canary survived → safe to continue.

v2 is your canary. 1% of users are the test.
```

```
CANARY DEPLOYMENT STAGES:

Stage 1:  1% → v2,  99% → v1    Watch for 30 min
Stage 2: 10% → v2,  90% → v1    Watch for 1 hour
Stage 3: 50% → v2,  50% → v1    Watch for 2 hours
Stage 4:100% → v2,   0% → v1    Full rollout complete

At any stage: metrics look bad → roll back to 100% v1.
```

---

### What Metrics You Watch

The moment you shift traffic to the canary, you watch **four golden signals** comparing v1 vs v2:

```
┌────────────────────┬──────────────────┬──────────────────┐
│ Signal             │ v1 (baseline)    │ v2 (canary)      │
├────────────────────┼──────────────────┼──────────────────┤
│ Error rate         │ 0.1%             │ 0.1%   ✅ same   │
│ p99 latency        │ 240ms            │ 238ms  ✅ same   │
│ Success rate       │ 99.9%            │ 99.9%  ✅ same   │
│ CPU usage          │ 45%              │ 46%    ✅ same   │
└────────────────────┴──────────────────┴──────────────────┘
→ Canary is healthy. Promote to 10%.

┌────────────────────┬──────────────────┬──────────────────┐
│ Signal             │ v1 (baseline)    │ v2 (canary)      │
├────────────────────┼──────────────────┼──────────────────┤
│ Error rate         │ 0.1%             │ 4.2%   ❌ spike  │
│ p99 latency        │ 240ms            │ 890ms  ❌ slow   │
│ Success rate       │ 99.9%            │ 95.8%  ❌ drop   │
└────────────────────┴──────────────────┴──────────────────┘
→ Canary is unhealthy. Roll back immediately.
  Only 1% of users were affected.
```

---

### Canary vs Blue-Green Side by Side

```
BLUE-GREEN:                        CANARY:
                                   
v1: ████████████████ 100%          v1: ███████████████░  99%
v2: ░░░░░░░░░░░░░░░░   0%          v2: █                  1%
         │                                  │
    ONE flip                         gradual shift
         │                                  │
v1: ░░░░░░░░░░░░░░░░   0%          v1: ░░░░░░░░░░░░░░░░   0%
v2: ████████████████ 100%          v2: ████████████████ 100%

All-or-nothing.                    Incremental confidence building.
Instant rollback.                  Catch bugs before full exposure.
Best for: high-risk schema         Best for: feature releases,
changes, all-or-nothing            behavior changes, A/B testing,
correctness requirements.          performance validation.
```

---

### Implementing Canary in Kubernetes

#### Without Service Mesh — Pod Ratio Trick

K8s Service load balances across ALL pods matching the selector proportionally:

```yaml
# v1 Deployment — 9 pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v1
spec:
  replicas: 9                    # ← 9 pods = 90% of traffic
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    spec:
      containers:
        - image: shopsphere/order-service:v1.0
---
# v2 Canary Deployment — 1 pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-v2-canary
spec:
  replicas: 1                    # ← 1 pod = ~10% of traffic
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v2              # different version label
    spec:
      containers:
        - image: shopsphere/order-service:v2.0
---
# Service selects ALL pods with app=order-service
# regardless of version label
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service           # ← selects BOTH v1 and v2 pods
```

```
Traffic distribution:
  10 total pods → K8s round-robins across all 10
  9 pods are v1 → ~90% of requests hit v1
  1 pod is v2  → ~10% of requests hit v2

Increase canary:
  Scale v2 to 3 pods, scale v1 to 7 pods → 30% canary
  Scale v2 to 5 pods, scale v1 to 5 pods → 50% canary

Limitation: coarse-grained, pod-count-based only.
Can't target specific users or headers.
```

---

#### With Istio — Precise Percentage Control

No need to manage pod counts. Istio splits traffic at the proxy layer:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 99             # ← 99% to v1
        - destination:
            host: order-service
            subset: v2
          weight: 1              # ← 1% to v2
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

Promote canary — just change weights. No pod scaling needed:

```yaml
# After 30 min healthy → promote to 10%
weight: 90  # v1
weight: 10  # v2

# After 1 hour healthy → promote to 50%
weight: 50  # v1
weight: 50  # v2

# Full rollout
weight: 0   # v1
weight: 100 # v2
```

---

### Header-Based Canary — Targeted Routing

Instead of random 1%, route specific users to the canary:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    # Internal QA team always hits canary
    - match:
        - headers:
            x-canary-user:
              exact: "true"
      route:
        - destination:
            host: order-service
            subset: v2
          weight: 100

    # Beta users by cookie
    - match:
        - headers:
            cookie:
              regex: ".*beta_user=true.*"
      route:
        - destination:
            host: order-service
            subset: v2
          weight: 100

    # Everyone else → v1
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 100
```

```
Internal engineer sends request with header x-canary-user: true
  → Always hits v2 canary
  → Can thoroughly test in production with real data

Beta users (opted in) hit v2
Regular users still hit v1 safely
```

---

### Automated Canary — Argo Rollouts

Manually watching metrics and updating weights gets tedious. **Argo Rollouts** automates this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: order-service
spec:
  strategy:
    canary:
      steps:
        - setWeight: 1          # Step 1: send 1% to canary
        - pause: {duration: 30m}# Wait 30 minutes
        - setWeight: 10         # Step 2: send 10%
        - pause: {duration: 1h} # Wait 1 hour
        - setWeight: 50         # Step 3: send 50%
        - pause: {duration: 2h} # Wait 2 hours
        - setWeight: 100        # Full rollout

      # Auto-rollback if metrics breach thresholds
      analysis:
        templates:
          - templateName: order-service-analysis
        startingStep: 1

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: order-service-analysis
spec:
  metrics:
    - name: error-rate
      interval: 5m
      successCondition: result[0] < 0.01   # error rate < 1%
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              app="order-service",
              status=~"5.."
            }[5m])) /
            sum(rate(http_requests_total{
              app="order-service"
            }[5m]))

    - name: p99-latency
      successCondition: result[0] < 500    # p99 < 500ms
      provider:
        prometheus:
          query: |
            histogram_quantile(0.99,
              rate(http_request_duration_ms_bucket{
                app="order-service"
              }[5m]))
```

```
Argo Rollouts behavior:
  Deploy v2 → automatically sets weight to 1%
  Waits 30 minutes, queries Prometheus every 5 min
  Error rate 0.08% → under 1% threshold ✅
  Automatically advances to 10%... continues

  At 50% stage:
  Error rate spikes to 3.2% → exceeds 1% threshold
  FailureLimit 3 reached (3 consecutive bad checks)
  → Argo AUTOMATICALLY rolls back to 100% v1
  → Sends alert to Slack/PagerDuty
  → Zero human intervention needed
```

---

### Canary + Feature Flags Together

```
Canary controls WHICH VERSION of the binary users hit.
Feature flags control WHICH FEATURES are enabled within that version.

v2 deployed to 10% of users via canary.
But within v2, new checkout flow is feature-flagged OFF by default.

You can:
  Enable new checkout for 1% of v2 users (0.1% of total)
  Measure conversion rate vs old checkout
  If better → gradually enable for more v2 users
  If worse  → disable flag, no rollback needed

This is A/B testing layered on top of canary.
Next topic (Feature Flags) covers this in detail.
```

---

### The Soak Time Question

How long do you wait at each stage?

```
Too short:
  Promote 1% → 10% after 5 minutes
  Some bugs only appear after sustained load
  Or only on specific time-of-day patterns
  → You miss them

Too long:
  Sit at 1% for 1 week
  Slows down release velocity
  Team frustration

Practical guidance:
  Routine feature release:    30 min → 2h → 24h
  High-risk change:           1h → 6h → 24h → 72h
  Payment/auth change:        manual approval gates at each stage
  
  Always watch at least one full peak traffic period
  (e.g., if you deploy at 2am, soak through morning peak)
```

---

### ShopSphere Lens

```
ShopSphere canary strategy per service tier:

Tier 1 (money/auth — max caution):
  order-service, payment-service, user-service
  → Argo Rollouts with automated analysis
  → 1% → 5% → 20% → 100%
  → Each stage gated by Prometheus error rate + latency
  → Manual approval required at 20% → 100%

Tier 2 (customer-facing but recoverable):
  product-service, search-service, review-service
  → Istio weight-based canary
  → 10% → 50% → 100%
  → Automated promotion if metrics clean

Tier 3 (internal, low risk):
  notification-service
  → Standard rolling update
  → No canary needed (failures are non-critical)

GitHub Actions integration:
  On merge to main:
    1. Build + push image
    2. kubectl argo rollouts set image order-service v2.0
    3. Argo handles staged rollout automatically
    4. Pipeline monitors rollout status
    5. Fails pipeline if Argo auto-rolls back
       → PR author gets notified immediately
```

---

### Interview Questions

**Q1. What is a canary deployment and how does it differ from Blue-Green?**

> Canary gradually shifts a small percentage of real production traffic to the new version, watching metrics before expanding. Blue-Green flips 100% of traffic at once with no gradual exposure. Canary is better for catching bugs that only appear under real user behavior or at production scale — the blast radius of a bad deploy is limited to the canary percentage. Blue-Green is better when you need atomic, version-clean transitions with instant rollback.

**Q2. What metrics do you watch during a canary deployment?**

> The four golden signals: error rate (5xx responses compared between v1 and v2), latency (p99 compared between versions), success rate, and resource saturation (CPU/memory). The canary is compared against the stable baseline — if error rate or latency diverges beyond a threshold, the canary is rolled back automatically. You need consistent labeling (version tags) in your metrics to separate canary from stable traffic.

**Q3. How does Istio enable more precise canary control than native Kubernetes?**

> Native K8s canary works by pod ratio — 1 canary pod out of 10 total gives ~10% traffic. You can't do 1% without spinning up 99 stable pods, and you can't target specific users. Istio VirtualService splits traffic by weight at the proxy layer — you specify exact percentages regardless of pod count. You can also route based on request headers or cookies, enabling targeted canary for specific users like internal testers or opted-in beta users.

**Q4. What is automated canary analysis and why does it matter?**

> Manually watching dashboards during every canary promotion doesn't scale. Tools like Argo Rollouts query Prometheus metrics automatically at each stage — if error rate or latency exceeds defined thresholds for N consecutive checks, it triggers an automatic rollback. This removes human error from the promotion decision, enables canary deployments at any time of day without someone watching, and provides a consistent, auditable promotion policy across all services.

---

Ready for **Topic 10 — Feature Flags** — the last topic in Stage 8 — when you are.
