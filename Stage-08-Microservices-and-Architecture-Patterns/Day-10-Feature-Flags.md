## Topic 10 — Feature Flags

### The Problem First

Traditional release cycle tightly couples two things:

```
DEPLOY  =  RELEASE

You merge code → CI builds → deploys to prod → feature is LIVE
If feature has a bug → rollback the entire deployment
If feature needs to be hidden → revert the code

Problems this creates:
  1. Long-lived feature branches → merge conflicts nightmare
  2. "Deploy on Friday" anxiety → what if it breaks?
  3. Can't test in production with real data before go-live
  4. Can't release to specific users (beta, internal, region)
  5. Rollback = full redeployment → slow, risky
```

Feature Flags break this coupling entirely.

---

### What is a Feature Flag?

A **runtime configuration switch** that controls whether a feature is active — completely independent of whether the code is deployed.

```
Code is deployed to production.
Feature is OFF by default.
You flip a flag → feature turns ON for specific users.
No deployment. No code change. Just a config update.

DEPLOY  ≠  RELEASE

Deploy:  push code to production (technical event)
Release: make feature available to users (business event)

These are now two separate, independent decisions.
```

---

### The Core Concept

```java
// Without feature flag — feature is always live after deploy
public OrderResponse placeOrder(OrderRequest request) {
    return processOrder(request);
}

// With feature flag — controlled at runtime
public OrderResponse placeOrder(OrderRequest request) {
    if (featureFlagService.isEnabled("new-checkout-flow", request.getUserId())) {
        return newCheckoutFlow(request);    // new code path
    } else {
        return legacyCheckoutFlow(request); // old code path
    }
}
```

```
Flag OFF (default):   all users → legacyCheckoutFlow()
Flag ON for 1%:       1% users  → newCheckoutFlow()
                      99% users → legacyCheckoutFlow()
Flag ON for all:      all users → newCheckoutFlow()
Flag OFF again:       all users → legacyCheckoutFlow() ← instant rollback
```

No redeployment at any of these stages.

---

### Types of Feature Flags

Different flags serve different purposes with different lifetimes:

#### 1. Release Flags (most common)

Hide incomplete or unvalidated features in production.

```
Purpose: decouple deploy from release
Lifetime: short — days to weeks
Example:  new-checkout-flow, redesigned-product-page

Ship the code hidden. Validate in prod. Flip when ready.
Remove the flag once fully rolled out — it's tech debt otherwise.
```

#### 2. Experiment Flags (A/B Testing)

Expose two variants to different user segments. Measure which performs better.

```
Purpose: data-driven product decisions
Lifetime: medium — days to weeks until winner decided
Example:  checkout-button-color, recommendation-algorithm-v2

Group A (50%): sees green "Buy Now" button
Group B (50%): sees orange "Buy Now" button

Measure: conversion rate per group
Winner: orange button → 12% higher conversion
Decision: ship orange to 100%, remove flag
```

#### 3. Ops Flags (Kill Switches)

Disable expensive or risky features under system stress.

```
Purpose: system resilience, graceful degradation
Lifetime: permanent — never removed
Example:  enable-recommendations, enable-search-suggestions
          enable-real-time-inventory-check

Black Friday peak load → CPU at 95%
Flip: enable-recommendations = false
  → recommendation engine calls disabled
  → load drops 20%
  → critical checkout flow protected

Flip back once load normalizes.
These flags stay in the codebase forever as emergency levers.
```

#### 4. Permission Flags

Enable features for specific user segments only.

```
Purpose: targeted rollout, beta programs, entitlements
Lifetime: permanent
Example:  premium-analytics-dashboard, bulk-export-feature
          early-access-new-ui

Beta users:    enable-new-ui = true
Free tier:     enable-bulk-export = false
Premium tier:  enable-bulk-export = true
Internal team: enable-all-experimental = true
```

---

### Flag Evaluation — The Decision Matrix

Flags aren't just ON/OFF globally. They evaluate **context**:

```
Flag: new-checkout-flow

Evaluation rules (checked in order):
  1. Is user in internal-testers group?    → ON  (always test internally first)
  2. Is user in beta-program?              → ON  (opted-in beta users)
  3. Is user in canary-1-percent?          → ON  (random 1% bucket)
  4. Is request from region = "IN"?        → OFF (not rolled out in India yet)
  5. Default                               → OFF

Request from internal engineer:
  Rule 1 matches → flag is ON → new checkout flow

Request from random user in US:
  Rules 1,2 skip → Rule 3: hash(userId) % 100 < 1? 
  userId=12345 → 12345 % 100 = 45 → 45 < 1? No → skip
  Rule 4: region=US, skip
  Default: OFF → legacy checkout flow

Request from random user in US (different userId):
  userId=99900 → 99900 % 100 = 0 → 0 < 1? Yes → Rule 3 matches → ON
```

This evaluation happens **in milliseconds in memory** — no DB query per request.

---

### Flag Storage and Evaluation Architecture

```
                        ┌─────────────────────┐
                        │   Flag Management   │
                        │   Dashboard (UI)    │
                        │   LaunchDarkly /    │
                        │   Unleash / custom  │
                        └──────────┬──────────┘
                                   │ flag config
                                   ▼
                        ┌─────────────────────┐
                        │   Flag Store        │
                        │   (Redis / DB)      │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │ order-service   │  │ product-service │  │ user-service    │
    │                 │  │                 │  │                 │
    │ SDK caches      │  │ SDK caches      │  │ SDK caches      │
    │ flags locally   │  │ flags locally   │  │ flags locally   │
    │ (in-memory)     │  │ (in-memory)     │  │ (in-memory)     │
    └─────────────────┘  └─────────────────┘  └─────────────────┘

SDK polls or receives push updates every 30s.
Flag evaluation = in-memory hashmap lookup.
Zero network call per request.
Flag store goes down → services use last cached state.
```

---

### The Tools

#### LaunchDarkly

The industry standard commercial solution. Used by Netflix, Atlassian, IBM.

```
Features:
  - Dashboard to manage all flags
  - SDK for every language (Java, Go, Python, Node, etc.)
  - Targeting rules (user attributes, segments, percentages)
  - Experimentation (A/B testing with stats engine)
  - Flag audit log (who flipped what when)
  - Real-time streaming updates to SDKs

Spring Boot integration:
  <dependency>
    <groupId>com.launchdarkly</groupId>
    <artifactId>launchdarkly-java-server-sdk</artifactId>
    <version>7.0.0</version>
  </dependency>

LDUser user = LDUser.builder(userId)
    .email(userEmail)
    .custom("plan", "premium")
    .build();

boolean enabled = ldClient.boolVariation("new-checkout-flow", user, false);
```

#### Unleash

Open-source alternative. Self-hosted. Free. Production-grade.

```
Features:
  - All core flag types
  - Activation strategies (gradual rollout, userId, IP, custom)
  - Admin UI
  - Spring Boot starter available
  - Good choice if you can't use SaaS (data compliance)

Docker Compose:
  unleash:
    image: unleashorg/unleash-server:latest
    ports:
      - "4242:4242"
    environment:
      DATABASE_URL: postgres://unleash:password@postgres/unleash
```

#### Spring Boot + Custom Redis Flag Store

For ShopSphere — simple, no external SaaS:

```java
@Service
public class FeatureFlagService {

    private final RedisTemplate<String, String> redisTemplate;

    // Check if flag is globally enabled
    public boolean isEnabled(String flagName) {
        String value = redisTemplate.opsForValue()
            .get("feature:" + flagName);
        return "true".equals(value);
    }

    // Check with user-level targeting
    public boolean isEnabled(String flagName, String userId) {
        // 1. Check user-specific override
        String userOverride = redisTemplate.opsForValue()
            .get("feature:" + flagName + ":user:" + userId);
        if (userOverride != null) return "true".equals(userOverride);

        // 2. Check percentage rollout
        String pctValue = redisTemplate.opsForValue()
            .get("feature:" + flagName + ":pct");
        if (pctValue != null) {
            int pct = Integer.parseInt(pctValue);
            int userBucket = Math.abs(userId.hashCode()) % 100;
            return userBucket < pct;
        }

        // 3. Global flag state
        return isEnabled(flagName);
    }
}
```

```
Redis keys:
  feature:new-checkout-flow          = "false"   (globally off)
  feature:new-checkout-flow:pct      = "10"      (10% of users)
  feature:new-checkout-flow:user:u99 = "true"    (always on for this user)
```

---

### Feature Flags + Canary Together

These two patterns compose naturally:

```
Canary:         controls which binary (v1 or v2) a request hits
Feature Flag:   controls which code path runs WITHIN that binary

Layer 1 — Canary (Istio weight):
  10% of requests → order-service v2
  90% of requests → order-service v1

Layer 2 — Feature Flag (within v2):
  new-checkout-flow = 50% of users within v2

Effective exposure:
  10% (canary) × 50% (flag) = 5% of all users see new checkout

This gives extremely fine-grained control:
  New binary deployed safely via canary
  New feature inside binary rolled out separately via flag
  Two independent rollback mechanisms
```

---

### Trunk-Based Development

Feature flags enable a practice called **trunk-based development**:

```
WITHOUT flags (long-lived branches):
  feature/new-checkout ──────────────────────────────┐
                                                      │ merge
  main ──────────────────────────────────────────────▶│
       ↑                                              │
       3 weeks of divergence = massive merge conflict

WITH flags (trunk-based):
  Developer works in small commits directly on main.
  New checkout code ships behind flag = OFF.
  No branch lives longer than 1-2 days.
  
  main ──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶──▶
         each commit small, deployable, safe

  No merge conflicts. Continuous integration is actual CI.
  Flag controls release, not branch.
```

---

### The Flag Lifecycle — Technical Debt Warning

```
Flags are temporary by design (except ops/permission flags).
Uncleaned flags become a serious maintenance problem:

After 6 months of ignoring flag cleanup:
  codebase has 47 active flags
  developers don't know which are safe to remove
  testing matrix explodes: 2^47 possible flag combinations
  code is riddled with if/else branches
  new developer reads code: "what does this flag even do?"

DISCIPLINE REQUIRED:

  1. Every flag has an owner and expiry date in JIRA/ticket
  2. Release flags → remove within 2 weeks of full rollout
  3. Experiment flags → remove within 1 week of decision
  4. Flag removal = PR that deletes the branch, not just the flag

  // Remove flag: delete this entire block
  if (featureFlagService.isEnabled("new-checkout-flow", userId)) {
      return newCheckoutFlow(request);  // ← this becomes the only path
  } else {
      return legacyCheckoutFlow(request); // ← delete this
  }
  // After cleanup:
  return newCheckoutFlow(request); // clean, no flag
```

---

### ShopSphere Lens

```
ShopSphere feature flags strategy:

Ops flags (permanent kill switches):
  enable-elasticsearch-search      (fallback to DB search if ES is down)
  enable-recommendation-engine     (disable if CPU spikes)
  enable-email-notifications       (disable if SendGrid quota hit)
  enable-real-time-stock-check     (disable under peak load)

Release flags (temporary):
  new-review-ranking-algorithm     (testing new ML-based ranking)
  bulk-order-api-v2                (new B2B bulk order endpoint)
  enhanced-product-search-filters  (new filter UX)

Permission flags (permanent):
  premium-analytics-dashboard      (only premium merchants)
  api-rate-limit-relaxed           (enterprise tier only)

Implementation in ShopSphere:
  Redis already in stack (session cache, rate limiting)
  → FeatureFlagService backed by Redis (no new infra)
  → @ConditionalOnFeatureFlag custom annotation for Spring beans
  → Flag state visible in Actuator custom endpoint /actuator/flags
  → Flip flags via admin API secured behind ADMIN role

Example in product-service:
  GET /api/products/search?q=shoes

  if (flags.isEnabled("enable-elasticsearch-search", userId)) {
      return elasticsearchRepository.search(q);  // fast
  } else {
      return productRepository.searchByName(q);  // DB fallback
  }
```

---

### Interview Questions

**Q1. What is a feature flag and how does it decouple deployment from release?**

> A feature flag is a runtime configuration switch that controls feature availability independently of code deployment. Code ships to production with the feature disabled. When ready, the flag is flipped — no redeployment needed. This means developers can merge small, incremental commits to main continuously, ship to production safely, and control exactly when and for whom a feature goes live, with instant rollback by simply toggling the flag off.

**Q2. What are the different types of feature flags and when do you use each?**

> Release flags are short-lived, used to hide incomplete features during development. Experiment flags enable A/B testing — splitting users into groups to measure which variant performs better. Ops flags are permanent kill switches for graceful degradation under load — disabling expensive features like recommendations when the system is under stress. Permission flags are permanent, controlling feature access based on user tier or role.

**Q3. What is the biggest risk with feature flags and how do you manage it?**

> Flag debt — accumulation of stale flags that nobody removes. With 50 flags in the codebase, the testing matrix explodes, code becomes hard to read, and developers lose confidence about what's safe to remove. Mitigation: every flag has an owner and expiry date, release and experiment flags are deleted within days of full rollout, and flag removal is treated as a required follow-up task tracked in the same ticket as the feature.

**Q4. How do feature flags and canary deployments complement each other?**

> They operate at different layers. Canary controls which binary version a request hits — it's infrastructure-level traffic splitting. Feature flags control which code path runs within a binary — it's application-level. Together they provide two independent rollback mechanisms: roll back the canary if the binary itself is broken, flip the flag if the specific feature behavior is broken but the rest of the binary is healthy. They also compose — canary limits blast radius of the binary, flags limit blast radius of individual features within it.

---

