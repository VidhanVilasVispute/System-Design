# Stage 2 — Topic 10: Authentication vs Authorization

## Theory

Authentication and Authorization are two of the most confused terms in software engineering. They are different problems, solved by different mechanisms, at different layers of your system. Mixing them up in an interview is a red flag.

**Authentication (AuthN) — Who are you?**
The process of verifying that someone is who they claim to be. Proving identity.

**Authorization (AuthZ) — What are you allowed to do?**
The process of determining whether an authenticated identity has permission to perform a specific action on a specific resource.

```
Real world analogy:

Authentication:
  You show your passport at the airport.
  The officer verifies it is genuine and matches your face.
  Identity confirmed — they know WHO you are.

Authorization:
  You board the plane and try to sit in Business Class.
  The flight attendant checks your boarding pass.
  Your ticket is Economy — you are not ALLOWED to sit there.
  Identity was already known — this is about PERMISSION.
```

**The order is always fixed — AuthN first, AuthZ second:**
```
Request arrives
     ↓
Authentication — verify identity
     ↓ (reject with 401 if identity cannot be verified)
Authorization — check permissions
     ↓ (reject with 403 if identity lacks permission)
Business logic
```

You cannot authorise someone whose identity you have not verified. And verifying identity does not automatically grant permissions.

---

## Authentication — Deep Dive

### What Authentication Proves

Authentication answers: **"Is this entity who they claim to be?"**

Three factors of authentication — something you:
```
Know:    Password, PIN, security question answer
Have:    OTP device, authenticator app, hardware key (YubiKey)
Are:     Fingerprint, face recognition, voice (biometrics)

Multi-Factor Authentication (MFA):
  Requires two or more factors from different categories
  Password (know) + OTP (have) = 2FA
  Much stronger — attacker needs both factors
```

### Authentication Mechanisms

**1. Session-Based Authentication (stateful):**
```
Client                    Server                    Session Store
   |                         |                          |
   |--- POST /login -------->|                          |
   |    username+password    | validates credentials    |
   |                         |--- store session ------->|
   |                         |    sessionId: abc123     |
   |<-- Set-Cookie: ----------|    userId: u-123        |
   |    sessionId=abc123      |    expiresAt: ...       |
   |                         |                          |
   |--- GET /orders -------->|                          |
   |    Cookie: abc123       |--- lookup session ------>|
   |                         |<-- userId: u-123 --------|
   |<-- 200 OK orders -------|                          |

Server must store session state — breaks horizontal scaling
Every request hits the session store (Redis) — extra hop
Session invalidation is instant — just delete from store
```

**2. Token-Based Authentication (stateless):**
```
Client                    Server
   |                         |
   |--- POST /login -------->|
   |    username+password    | validates credentials
   |                         | generates JWT token
   |<-- JWT token -----------|  (self-contained, signed)
   |                         |
   |--- GET /orders -------->|
   |    Authorization:       | validates JWT signature
   |    Bearer <JWT>         | reads userId from token
   |<-- 200 OK orders -------|  (no session store needed)

No server-side state — stateless, scales horizontally
No session store lookup — just cryptographic verification
Token invalidation is hard — cannot revoke until expiry
```

**3. API Key Authentication:**
```
Client sends in header:
  X-API-Key: sk_live_abc123xyz

Server:
  Looks up API key in database
  Retrieves associated account and permissions
  Proceeds if key is valid and active

Use case: machine-to-machine authentication
  Third-party developers accessing ShopSphere API
  Internal service-to-service calls
  Webhooks and integrations

Never in browsers — API keys are long-lived secrets,
easy to accidentally expose in client-side code
```

**4. Certificate-Based Authentication (mTLS):**
```
Standard TLS:
  Client verifies SERVER certificate (one-way)
  Server trusts all clients

Mutual TLS (mTLS):
  Client verifies server certificate
  Server ALSO verifies CLIENT certificate
  Both sides authenticated cryptographically

Use case: internal microservice authentication
  Order Service presents its certificate
  Payment Service verifies it — "this is genuinely Order Service"
  No JWT, no API key — certificate IS the identity

ShopSphere: Istio service mesh can enforce mTLS
between all microservices automatically
```

---

## Authorization — Deep Dive

### What Authorization Decides

Authorization answers: **"Is this authenticated identity allowed to perform this action on this resource?"**

Three things in every authorization decision:
```
Subject:  WHO is asking           → user u-123, role ADMIN, service order-service
Action:   WHAT they want to do    → READ, WRITE, DELETE, EXECUTE
Resource: WHAT they want to do it to → /orders/o-789, /admin/users, /products/p-456
```

### Authorization Models

**1. ACL — Access Control List:**
```
Each resource has a list of who can access it:

Order o-789:
  u-123: READ, CANCEL
  admin-001: READ, WRITE, DELETE, CANCEL

Simple but does not scale:
  1 million orders × average 3 users each = 3 million ACL entries
  Adding a new permission requires updating every resource's ACL
```

**2. RBAC — Role-Based Access Control:**
```
Users are assigned roles. Roles have permissions.

Roles in ShopSphere:
  CUSTOMER:
    - Read own orders
    - Place orders
    - Cancel own orders (within window)
    - Read public products

  SELLER:
    - All CUSTOMER permissions
    - Create/update own products
    - View own sales analytics
    - Manage own inventory

  ADMIN:
    - All SELLER permissions
    - Read/update ANY order
    - Manage ALL products
    - Access user management
    - View system analytics

  SUPER_ADMIN:
    - All ADMIN permissions
    - Manage roles and permissions
    - System configuration

User u-123 has role CUSTOMER
User u-456 has role SELLER
User u-789 has role ADMIN
```

```java
// Spring Security RBAC implementation
@RestController
public class OrderController {

    @GetMapping("/orders/{orderId}")
    @PreAuthorize("hasRole('ADMIN') or " +
                  "@orderSecurityService.isOwner(#orderId, authentication)")
    public OrderResponse getOrder(@PathVariable String orderId) {
        return orderService.getOrder(orderId);
    }

    @DeleteMapping("/admin/orders/{orderId}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable String orderId) {
        orderService.delete(orderId);
    }

    @PostMapping("/orders")
    @PreAuthorize("hasAnyRole('CUSTOMER', 'SELLER', 'ADMIN')")
    public OrderResponse createOrder(@RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }
}

// Fine-grained ownership check
@Service
public class OrderSecurityService {
    public boolean isOwner(String orderId, Authentication auth) {
        String userId = extractUserId(auth);
        return orderRepository.existsByIdAndUserId(orderId, userId);
    }
}
```

**3. ABAC — Attribute-Based Access Control:**
```
Permissions based on attributes of subject, resource, action, and environment.
More flexible than RBAC — policies expressed as rules:

Policy 1: Allow access if
  subject.role == SELLER AND
  resource.sellerId == subject.userId AND
  action == READ OR WRITE

Policy 2: Allow access if
  subject.role == CUSTOMER AND
  resource.userId == subject.userId AND
  action == READ

Policy 3: Allow access if
  subject.role == ADMIN AND
  environment.time BETWEEN 09:00 AND 18:00  ← time-based
  (admin access only during business hours)

ABAC is powerful but complex to implement and audit.
Used in enterprise systems with complex permission requirements.
```

**4. ReBAC — Relationship-Based Access Control:**
```
Permissions based on relationships between entities.
Used by Google (Zanzibar), GitHub, Notion.

"User u-123 can access document d-456 if
  u-123 is a member of team t-789 which owns d-456"

Relationships:
  u-123 → member of → team t-789
  team t-789 → owner of → document d-456
  
  Therefore: u-123 can access document d-456

ShopSphere application:
  u-123 → buyer of → order o-789
  u-456 → seller of → product p-999 which is in order o-789
  
  u-456 can see order o-789's status but not payment details
  u-123 can cancel o-789 but only within 30 minutes
```

---

## Where Authentication and Authorization Live in ShopSphere

```
Request flow:

Client Request
      ↓
API Gateway (Spring Cloud Gateway)
  ├── Authentication:
  │     Extract JWT from Authorization header
  │     Verify JWT signature with public key
  │     Check JWT expiry
  │     Reject with 401 if invalid/missing
  │     Extract userId, roles, sessionId from claims
  │     Forward userId + roles in internal headers
  │
  └── Coarse-grained Authorization:
        Is this role allowed to access this path at all?
        CUSTOMER cannot access /admin/* → 403
        Forward if passed
              ↓
      Microservice (e.g. Order Service)
        Fine-grained Authorization:
          Does this CUSTOMER own this order?
          Can this SELLER modify this product?
          @PreAuthorize checks at method level
              ↓
          Business Logic
```

**Why split authorization between gateway and service:**
```
API Gateway handles:
  Path-level access control
  Role-based route restrictions
  DDoS protection — reject bad requests early
  Fast — no DB lookup needed, just JWT claims

Microservice handles:
  Resource-level ownership checks
  Fine-grained business rules
  "Can user X perform action Y on resource Z?"
  Requires DB lookup — does this user own this order?
```

---

## Common Authorization Mistakes

**Mistake 1 — Trusting client-supplied identity:**
```java
// DANGEROUS — never do this
@GetMapping("/orders")
public List<Order> getOrders(@RequestParam String userId) {
    return orderService.getOrdersForUser(userId);
    // Attacker passes userId=u-001 and sees admin's orders
}

// CORRECT — extract identity from verified JWT
@GetMapping("/orders")
public List<Order> getOrders(@AuthenticationPrincipal UserDetails user) {
    return orderService.getOrdersForUser(user.getId());
    // userId comes from verified JWT — cannot be spoofed
}
```

**Mistake 2 — Missing resource-level checks:**
```java
// VULNERABLE — checks authentication but not ownership
@GetMapping("/orders/{orderId}")
public OrderResponse getOrder(@PathVariable String orderId) {
    return orderService.getOrder(orderId);
    // Any authenticated user can see any order by ID
    // Insecure Direct Object Reference (IDOR) vulnerability
}

// SECURE — checks ownership
@GetMapping("/orders/{orderId}")
public OrderResponse getOrder(@PathVariable String orderId,
                               @AuthenticationPrincipal UserDetails user) {
    Order order = orderService.getOrder(orderId);
    if (!order.getUserId().equals(user.getId()) && !user.hasRole("ADMIN")) {
        throw new AccessDeniedException("Not your order");
    }
    return mapToResponse(order);
}
```

**Mistake 3 — 401 vs 403 confusion:**
```
Return 401 when: identity is missing or invalid
  → "You need to log in"
  → WWW-Authenticate header should be present

Return 403 when: identity is valid but lacks permission
  → "You are logged in but not allowed to do this"
  → Do NOT redirect to login — they are already logged in

Common mistake: returning 404 instead of 403
  Sometimes done deliberately to avoid revealing that
  a resource exists — "security through obscurity"
  Debatable practice — use consistently if you choose it
```

**Mistake 4 — Overly broad roles:**
```
Bad role design:
  ADMIN can do everything in every service
  
Problems:
  Compromised admin account = total system breach
  No audit trail of what was done
  Violates principle of least privilege

Good role design:
  ORDER_ADMIN   — can manage orders only
  PRODUCT_ADMIN — can manage products only
  USER_ADMIN    — can manage users only
  SUPER_ADMIN   — can assign roles (rarely used)
  
Principle of least privilege:
  Every identity gets the minimum permissions needed
  to do its job — nothing more
```

---

## Service-to-Service Authentication

Services authenticating to other services is just as important as user authentication:

```
Option 1 — JWT with service identity:
  Order Service generates a JWT:
    sub: "order-service"
    roles: ["INTERNAL_SERVICE"]
  
  Sends it to Payment Service in Authorization header
  Payment Service verifies — this is a legitimate internal caller

Option 2 — API Keys:
  Each service has a long-lived API key
  Stored in secrets manager (Vault, AWS Secrets Manager)
  Rotated periodically
  
Option 3 — mTLS (strongest):
  Each service has a certificate
  Istio/Envoy handles cert rotation automatically
  No code changes needed — infrastructure-enforced

ShopSphere internal:
  Development: JWT service tokens
  Production:  mTLS via Istio service mesh
```

---

## Real-World Example — ShopSphere Security Flow

```java
// API Gateway security configuration
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(
            ServerHttpSecurity http) {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeExchange(auth -> auth
                // Public endpoints — no auth needed
                .pathMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .pathMatchers(HttpMethod.GET, "/api/v1/search/**").permitAll()
                .pathMatchers("/api/v1/auth/**").permitAll()
                .pathMatchers("/actuator/health").permitAll()

                // Admin endpoints — ADMIN role required
                .pathMatchers("/api/v1/admin/**").hasRole("ADMIN")

                // Seller endpoints — SELLER or ADMIN role
                .pathMatchers("/api/v1/seller/**").hasAnyRole("SELLER", "ADMIN")

                // All other endpoints — any authenticated user
                .anyExchange().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            )
            .build();
    }

    @Bean
    public ReactiveJwtAuthenticationConverter jwtAuthConverter() {
        ReactiveJwtAuthenticationConverter converter =
            new ReactiveJwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Extract roles from JWT claims
            List<String> roles = jwt.getClaimAsStringList("roles");
            return Flux.fromIterable(roles)
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role));
        });
        return converter;
    }
}

// Order Service — fine-grained authorization
@Service
public class OrderAuthorizationService {

    public void assertCanAccessOrder(String orderId, UserDetails user) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        boolean isOwner = order.getUserId().equals(user.getId());
        boolean isAdmin  = user.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
        boolean isSeller = order.getItems().stream()
            .anyMatch(item -> item.getSellerId().equals(user.getId()));

        if (!isOwner && !isAdmin && !isSeller) {
            throw new AccessDeniedException(
                "User " + user.getId() + " cannot access order " + orderId
            );
        }
    }
}
```

---

## Interview Q&A

**Q: What is the difference between authentication and authorization?**
Authentication verifies identity — who you are. It answers "is this person who they claim to be?" via passwords, tokens, certificates, or biometrics. Authorization determines permissions — what you are allowed to do. It answers "does this verified identity have permission to perform this action on this resource?" Authentication always happens first. A 401 response means authentication failed. A 403 response means authentication succeeded but authorization failed.

**Q: What is the difference between RBAC and ABAC?**
RBAC assigns permissions to roles and users to roles — simple, auditable, easy to reason about. A user with the ADMIN role can manage orders; a CUSTOMER cannot. ABAC grants access based on attributes of the subject, resource, action, and environment — far more expressive. A policy might allow a seller to edit a product only if the product belongs to that seller AND the request comes during business hours. RBAC is sufficient for most systems. ABAC is needed when role alone is insufficient to make access decisions — when context and resource attributes matter.

**Q: What is the principle of least privilege?**
Every identity — user, service, or process — should have the minimum permissions necessary to do its job and nothing more. A customer should not have admin permissions. The Order Service should not have write access to the User Service database. An API key for reading products should not be able to create orders. Least privilege limits the blast radius of a compromised credential — if an attacker gains access, they can only do what that identity was allowed to do, not everything.

**Q: What is an IDOR vulnerability and how do you prevent it?**
Insecure Direct Object Reference occurs when an API exposes a resource identifier — like an order ID — and relies only on authentication without checking ownership. An authenticated user who knows or guesses another user's order ID can access it. Prevention requires resource-level authorization — after authenticating the caller, verify they own or are permitted to access the specific resource. Never trust that authenticated users can only reach their own data unless you explicitly check it.

**Q: How do you handle service-to-service authentication in microservices?**
Three main approaches. JWT service tokens — each service includes a signed JWT identifying itself, verified by the receiving service. API keys stored in a secrets manager — simpler but requires key rotation management. Mutual TLS — each service has a certificate, both sides authenticate each other cryptographically — the strongest option. In production with a service mesh like Istio, mTLS is enforced transparently at the infrastructure layer with automatic certificate rotation, requiring no application code changes.

---

Say **"next"** when ready for Topic 10 — JWT & OAuth2.
