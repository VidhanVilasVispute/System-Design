## Topic 8 — Blue-Green Deployments

### The Problem First

Traditional deployment has a dangerous window:

```
You deploy new version of order-service:
  1. Stop old version (v1)         ← DOWNTIME STARTS
  2. Deploy new version (v2)
  3. Start new version (v2)        ← DOWNTIME ENDS

Even if steps 2-3 take 30 seconds → 30 seconds of 503s
In production with real users → unacceptable

Or rolling update approach:
  Kill pod 1 → start v2 pod 1 → kill pod 2 → start v2 pod 2...
  During transition: v1 and v2 both serve traffic simultaneously
  Problem: if v2 has a bug in order schema migration
           some users hit v1, some hit v2
           data inconsistency, mixed behavior
           hard to reason about
```

**Blue-Green Deployment** eliminates both problems.

---

### What is Blue-Green Deployment?

Maintain **two identical production environments** — Blue and Green. Only one serves live traffic at a time. Deploy to the idle one, then flip traffic instantly.

```
BLUE ENVIRONMENT (currently live, serving 100% traffic)
┌─────────────────────────────────────────┐
│  order-service v1  (3 pods)             │
│  product-service v1 (3 pods)            │
│  user-service v1   (3 pods)             │
└─────────────────────────────────────────┘
         ▲
         │ 100% traffic
         │
    Load Balancer / API Gateway
         │
         │ 0% traffic
         ▼
GREEN ENVIRONMENT (idle, ready)
┌─────────────────────────────────────────┐
│  order-service v2  (3 pods)             │
│  product-service v2 (3 pods)            │
│  user-service v2   (3 pods)             │
└─────────────────────────────────────────┘
```

---

### The Deployment Flow

#### Step 1 — Green is idle, Blue is live

```
Users ──────────────▶ Load Balancer ──────────────▶ BLUE (v1) ✅
                                    ──────────────▶ GREEN (v2) ❌ (no traffic)
```

#### Step 2 — Deploy v2 to Green

```
Deploy new code to Green environment.
Run DB migrations (additive only — covered later).
Run smoke tests, health checks against Green directly.
Users are still 100% on Blue — zero impact.

Users ──────────────▶ Load Balancer ──────────────▶ BLUE (v1) ✅
                                                    GREEN (v2) 🔧 deploying...
```

#### Step 3 — Test Green in isolation

```
QA team, automated tests hit Green directly via internal URL.
  http://green.internal.shopsphere.com/orders
  
Green passes all tests. Confidence is high.
Users still 100% on Blue. Nothing has changed for them.
```

#### Step 4 — Flip the switch

```
ONE config change at load balancer / API Gateway:
  route 100% traffic → GREEN instead of BLUE

Users ──────────────▶ Load Balancer ──────────────▶ BLUE (v1) ❌ (standby)
                                    ──────────────▶ GREEN (v2) ✅

This flip is instantaneous. Zero downtime. Zero degradation.
```

#### Step 5 — Blue becomes the rollback safety net

```
If Green (v2) starts throwing errors after the flip:
  ONE config change → route traffic back to Blue (v1)
  
  Rollback time: seconds, not minutes.
  
  Users ──▶ Load Balancer ──▶ BLUE (v1) ✅  ← instantly back
                              GREEN (v2) ❌  ← taken out of rotation
```

---

### Why This Works — The Key Properties

```
1. Zero downtime
   Traffic flip is a config change, not a deployment.
   Load balancer updates routing rules in milliseconds.

2. Instant rollback
   Blue is always warm and ready.
   No need to re-deploy v1 → it's already running.
   Rollback = flip traffic back. Takes seconds.

3. Clean separation
   v1 and v2 NEVER serve traffic simultaneously.
   No mixed-version state for users to hit.
   No version compatibility issues mid-deploy.

4. Production validation before go-live
   Green runs real production infrastructure.
   You test on identical config before any user sees it.
```

---

### The Database Problem

Blue-Green sounds simple until you add a database:

```
BLUE and GREEN share the same production DB
(you can't run two production DBs simultaneously — data would diverge)

┌────────────────┐          ┌────────────────┐
│  BLUE (v1)     │          │  GREEN (v2)    │
│  order-service │          │  order-service │
└───────┬────────┘          └───────┬────────┘
        │                           │
        └───────────┬───────────────┘
                    │
          ┌─────────▼──────────┐
          │  Shared Production │
          │      Database      │
          └────────────────────┘
```

This creates a constraint: **during the transition window, both v1 and v2 must be able to work with the same DB schema.**

```
DANGEROUS migration:
  v2 renames column: product_name → title

  While Green is being tested (Blue still live):
    Blue (v1) does: SELECT product_name FROM products → ❌ column gone
    Green (v2) does: SELECT title FROM products → ✅

  Blue breaks the moment you run the migration.
  You can't safely migrate and test simultaneously.
```

#### The solution: Expand/Contract Pattern (a.k.a. Parallel Change)

```
Phase 1 — EXPAND (deploy with Blue still live):
  Add NEW column: ALTER TABLE products ADD COLUMN title VARCHAR;
  v2 writes to BOTH product_name AND title
  v2 reads from title
  v1 still reads from product_name → works fine
  Both versions compatible with DB simultaneously ✅

Phase 2 — CONTRACT (after Blue is fully decommissioned):
  DROP COLUMN product_name
  v1 is gone, no risk anymore

Rule: DB migrations must always be ADDITIVE first.
Never rename or drop columns in the same deploy as the code change.
```

---

### Blue-Green in Kubernetes

In K8s, Blue-Green is implemented with **labels and Service selectors**:

```yaml
# Blue Deployment (v1) — currently live
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue       # ← label identifies this as blue
    spec:
      containers:
        - name: order-service
          image: shopsphere/order-service:v1.0
---
# K8s Service — points to blue via selector
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue           # ← ALL traffic goes to blue pods
  ports:
    - port: 8080
```

Deploy Green (v2) alongside Blue:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: green
  template:
    metadata:
      labels:
        app: order-service
        version: green      # ← separate label
    spec:
      containers:
        - name: order-service
          image: shopsphere/order-service:v2.0
```

**The flip** — one kubectl command:

```bash
# Switch Service selector from blue → green
kubectl patch service order-service \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Instant. Zero downtime. All new connections go to green.
# Rollback:
kubectl patch service order-service \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

### Blue-Green with Istio

With Istio, you get even more control:

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
            subset: green
          weight: 100
        - destination:
            host: order-service
            subset: blue
          weight: 0
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
    - name: blue
      labels:
        version: blue
    - name: green
      labels:
        version: green
```

Change weights → instant traffic shift. No pod restart. No Service spec change. Pure config.

---

### Cost — The Double Infrastructure Problem

```
Blue-Green requires running TWO full environments simultaneously.

ShopSphere: 22 services × 3 pods each = 66 pods (normal)
Blue-Green: 22 services × 3 pods Blue
          + 22 services × 3 pods Green
          = 132 pods during deployment window

Cost doubles during deployment.

Mitigation strategies:
  1. Scale Green to minimum pods for testing → scale up after flip
  2. Use Blue-Green only for critical services
     (order, payment) not for all 22
  3. Immediately decommission Blue after confidence period
     (don't keep it running indefinitely)
  4. Cloud auto-scaling → spin up Green on deploy,
     terminate Blue after 30 min with no issues
```

---

### Blue-Green vs Rolling Update vs Canary

```
┌──────────────────┬─────────────────────┬──────────────────────┐
│                  │ Blue-Green          │ Rolling Update       │
├──────────────────┼─────────────────────┼──────────────────────┤
│ Downtime         │ Zero                │ Zero                 │
│ Rollback speed   │ Seconds (flip back) │ Minutes (re-deploy)  │
│ Mixed versions   │ Never               │ Yes, during rollout  │
│ Resource cost    │ 2x during deploy    │ Slight overhead      │
│ DB migrations    │ Complex (expand)    │ Complex (expand)     │
│ Best for         │ High-risk releases  │ Routine updates      │
└──────────────────┴─────────────────────┴──────────────────────┘

Canary (next topic) sits between these two:
  Partial traffic to new version → gradual rollout
  Not all-or-nothing like Blue-Green
```

---

### ShopSphere Lens

```
ShopSphere critical services for Blue-Green:
  - order-service   (money involved, rollback must be instant)
  - payment-service (zero tolerance for v1/v2 mix)
  - user-service    (auth flows, session handling)

ShopSphere non-critical (rolling update is fine):
  - notification-service (fire-and-forget, stateless)
  - search-service       (Elasticsearch index always compatible)
  - review-service       (low risk changes)

GitHub Actions pipeline for Blue-Green:
  1. Build + push image (shopsphere/order-service:v2.0)
  2. Apply order-service-green Deployment YAML
  3. Wait for green pods to be Ready
  4. Run smoke test suite against green pods directly
  5. If tests pass → patch Service selector to green
  6. Monitor error rate for 15 minutes
  7. If clean → delete blue Deployment (save costs)
  8. If errors → patch Service back to blue (rollback)
```

---

### Interview Questions

**Q1. What is Blue-Green deployment and how does it achieve zero downtime?**

> Blue-Green maintains two identical production environments. Only one serves live traffic at a time. You deploy to the idle environment, test it in isolation, then switch the load balancer to route 100% traffic to the new version in one atomic config change — no pod restarts, no drain windows. Rollback is equally instant — flip traffic back to the old environment which is still running.

**Q2. What is the biggest challenge with Blue-Green deployments?**

> Database migrations. Both environments share the same production DB, so during the transition window, both old and new code must be schema-compatible simultaneously. The Expand/Contract pattern handles this — first add new columns alongside old ones (additive migration), deploy the new code that works with both, then drop old columns only after the old version is fully decommissioned.

**Q3. How do you implement Blue-Green in Kubernetes?**

> Deploy both versions as separate Deployments with distinct version labels. The K8s Service initially selects the blue pods via label selector. After deploying green and validating it, update the Service's selector to point to green — K8s immediately routes all new connections to green pods. Rollback is a single kubectl patch restoring the blue selector.

**Q4. When would you choose Blue-Green over a rolling update?**

> Blue-Green when you need instant rollback capability and cannot tolerate mixed-version traffic — typically for high-risk changes involving schema migrations, API contract changes, or payment/auth flows. Rolling updates are sufficient for routine, backward-compatible changes where the cost of running double infrastructure isn't justified.

---

Ready for **Topic 9 — Canary Deployments** when you are.
