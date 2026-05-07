# Stage 2 — Topic 11: JWT & OAuth2

## Theory

In the previous topic we established what authentication and authorization are. Now we go deep on the two most important mechanisms used to implement them in modern systems — **JWT** for stateless identity propagation and **OAuth2** for delegated authorization.

These are not alternatives to each other — they are complementary. OAuth2 is the **framework** that governs how authentication flows work. JWT is the **token format** that carries identity information. In practice they are almost always used together.

**The problem they solve:**

```
Old world — session-based auth:
  User logs in → server creates session → stores in Redis
  Every request → server looks up session → Redis hit
  
  Problems:
    Every request needs a Redis lookup — latency + cost
    Redis becomes a critical dependency — single point of failure
    Hard to share sessions across services
    Does not work for third-party access delegation

New world — JWT + OAuth2:
  User logs in → server issues JWT → client stores it
  Every request → client sends JWT → server verifies signature locally
  
  Benefits:
    No session store lookup — cryptographic verification only
    Stateless — any server can verify any token
    Self-contained — identity + permissions in the token itself
    Works across services and domains
```

---

## JWT — JSON Web Token

### Structure

A JWT is three Base64URL-encoded parts separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9    ← Header
.
eyJ1c2VySWQiOiJ1LTEyMyIsInJvbGVzIjpbIkNVU1RPTUVSIl0sImVtYWlsIjoidmlkaGFuQGV4YW1wbGUuY29tIiwiaWF0IjoxNzEyNTY2MjAwLCJleHAiOjE3MTI1NjcxMDB9    ← Payload
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c    ← Signature
```

**Part 1 — Header:**
```json
{
  "alg": "RS256",    ← signing algorithm
  "typ": "JWT"       ← token type
}
```

**Part 2 — Payload (Claims):**
```json
{
  "sub":    "u-123",                    ← subject (userId) — standard claim
  "iss":    "auth.shopsphere.com",      ← issuer — standard claim
  "aud":    "api.shopsphere.com",       ← audience — standard claim
  "iat":    1712566200,                 ← issued at (unix timestamp) — standard claim
  "exp":    1712567100,                 ← expiry (unix timestamp) — standard claim
  "jti":    "jwt-abc-123",              ← unique token ID — standard claim
  
  "email":  "vidhan@example.com",       ← custom claim
  "roles":  ["CUSTOMER"],               ← custom claim
  "name":   "Vidhan"                    ← custom claim
}
```

**Standard claims every JWT should have:**
```
sub  — who the token is about (userId)
iss  — who issued the token (your auth service)
aud  — who the token is intended for (your API)
iat  — when it was issued
exp  — when it expires
jti  — unique token ID (enables revocation tracking)
```

**Part 3 — Signature:**
```
The signature is created by:
  RSASHA256(
    base64url(header) + "." + base64url(payload),
    privateKey
  )

Verification uses the PUBLIC key:
  Any service with the public key can verify the signature
  Only the auth service with the PRIVATE key can create tokens
  
This is asymmetric signing — RS256 (RSA) or ES256 (ECDSA)
Symmetric signing — HS256 (HMAC) uses one shared secret
  Simpler but all services must share the secret — less secure at scale
```

---

### JWT Verification Flow

```java
// How any microservice verifies a JWT locally — no auth service call needed
@Component
public class JwtTokenValidator {

    // Public key fetched from auth service's JWKS endpoint on startup
    private final RSAPublicKey publicKey;

    public JwtClaims validate(String token) {
        try {
            // 1. Parse and verify structure
            JWTClaimsSet claims = SignedJWT.parse(token)
                .verify(new RSASSAVerifier(publicKey));  // verify signature

            // 2. Verify expiry
            if (claims.getExpirationTime().before(new Date())) {
                throw new TokenExpiredException("JWT has expired");
            }

            // 3. Verify issuer
            if (!claims.getIssuer().equals("auth.shopsphere.com")) {
                throw new InvalidTokenException("Unknown issuer");
            }

            // 4. Verify audience
            if (!claims.getAudience().contains("api.shopsphere.com")) {
                throw new InvalidTokenException("Wrong audience");
            }

            return mapClaims(claims);

        } catch (ParseException | JOSEException e) {
            throw new InvalidTokenException("JWT validation failed: " + e.getMessage());
        }
    }
}
```

**JWKS — JSON Web Key Set:**
```
Auth service exposes public keys at:
  GET https://auth.shopsphere.com/.well-known/jwks.json

Response:
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "key-2026-01",   ← key ID — for rotation
      "n":   "...",           ← public key modulus
      "e":   "AQAB"           ← public key exponent
    }
  ]
}

Benefits:
  Services fetch public keys on startup and cache them
  When keys rotate, services fetch new ones via kid lookup
  No shared secret needed — public key is truly public
  Microservices never need to call auth service per request
```

---

### JWT Token Strategy — Access Token + Refresh Token

A single token creates a dilemma:

```
Long-lived token (7 days):
  ✅ User stays logged in — good UX
  ❌ Compromised token valid for 7 days — security risk
  ❌ Cannot revoke without token blacklist — loses statelessness

Short-lived token (15 minutes):
  ✅ Compromised token expires quickly — more secure
  ❌ User logged out every 15 minutes — terrible UX
```

**Solution — two-token strategy:**

```
Access Token:
  Short-lived — 15 minutes to 1 hour
  Stateless — verified locally, no DB lookup
  Used for every API request
  When expired — use refresh token to get a new one

Refresh Token:
  Long-lived — 7 to 30 days
  Stateful — stored in DB, can be revoked
  Used ONLY to obtain new access tokens
  Stored in httpOnly cookie — not accessible to JavaScript
  Rotated on every use — each refresh issues a new refresh token
```

```
Flow:

1. Login:
   POST /auth/login { email, password }
   → Server verifies credentials
   → Issues: access_token (15min) + refresh_token (30days)
   → access_token in response body
   → refresh_token in httpOnly cookie

2. API calls (access token valid):
   GET /orders
   Authorization: Bearer <access_token>
   → Server verifies JWT locally
   → Returns orders

3. Access token expires:
   GET /orders → 401 Unauthorized

4. Client silently refreshes:
   POST /auth/refresh
   Cookie: refresh_token=<refresh_token>
   → Server verifies refresh token in DB
   → Issues new access_token + new refresh_token
   → Old refresh token invalidated

5. Logout:
   POST /auth/logout
   → Server deletes refresh token from DB
   → Access token expires naturally in 15 minutes
   → Client deletes access token from memory
```

**Refresh token rotation — security benefit:**
```
If a refresh token is stolen and used:
  Attacker uses stolen token → gets new tokens → old token invalidated
  
  When legitimate user tries to refresh with their (now invalidated) token:
  → Server detects reuse of invalidated token
  → Token family is compromised
  → ALL refresh tokens for this user are revoked
  → User must log in again
  
  Attacker's new tokens are also revoked.
  Attack window: minimal.
```

---

## OAuth2 — Open Authorization Framework

### What OAuth2 Solves

OAuth2 solves **delegated authorization** — allowing a third-party application to access resources on behalf of a user, without the user giving the third party their password.

```
Real world problem:

ShopSphere wants to let users sign in with Google
  (instead of creating a new username/password)

Without OAuth2:
  User gives ShopSphere their Google password
  ShopSphere can now do ANYTHING in their Google account
  If ShopSphere is breached, Google account is compromised
  Terrible idea.

With OAuth2:
  User logs into Google directly (never gives password to ShopSphere)
  Google issues ShopSphere a limited access token
  ShopSphere can read email address ONLY (scoped permission)
  User can revoke ShopSphere's access at any time
  Google password is never exposed
```

### OAuth2 Roles

```
Resource Owner:  The user — owns their data
Client:          The app requesting access — ShopSphere
Auth Server:     Issues tokens — Google, GitHub, your own auth service
Resource Server: Hosts protected resources — Google's API, your API
```

### OAuth2 Grant Types — The Four Flows

**1. Authorization Code Flow (most important — used for web/mobile apps):**

```
User's Browser          ShopSphere              Google Auth Server
      |                      |                         |
      |-- clicks              |                         |
      |   "Sign in            |                         |
      |    with Google" ----->|                         |
      |                       |                         |
      |<-- redirect to -------|                         |
      |  accounts.google.com  |                         |
      |  ?client_id=...       |                         |
      |  &redirect_uri=...    |                         |
      |  &scope=email,profile |                         |
      |  &state=abc123        |                         |
      |  &code_challenge=...  | ← PKCE (security)       |
      |                       |                         |
      |-------- login with Google credentials ---------->|
      |<-------- authorization CODE returned ------------|
      |
      |-- redirect to ShopSphere with code=XYZ123 ------>|
      |                       |                          |
      |                       |-- POST /token ---------->|
      |                       |   code=XYZ123            |
      |                       |   client_secret=...      |
      |                       |   code_verifier=...      |
      |                       |<-- access_token ---------|
      |                       |   refresh_token          |
      |                       |   id_token (OIDC)        |
      |<-- logged in ---------|
```

**Key security elements:**
```
state parameter:
  Random value generated by client
  Returned unchanged by auth server
  Client verifies it matches — prevents CSRF attacks

PKCE (Proof Key for Code Exchange):
  Client generates code_verifier (random string)
  Sends code_challenge = hash(code_verifier) in initial request
  Sends code_verifier when exchanging code for token
  Auth server verifies hash matches
  Prevents authorization code interception attacks
  Required for mobile apps, recommended for all

code vs token directly:
  Auth server returns a CODE not a token in the redirect
  Code is short-lived (60 seconds) and single-use
  ShopSphere backend exchanges code for token server-to-server
  Token never exposed in browser URL or history
```

**2. Client Credentials Flow (machine-to-machine):**

```
ShopSphere Service              Auth Server
       |                             |
       |-- POST /token ------------->|
       |   grant_type=client_credentials
       |   client_id=order-service
       |   client_secret=secret
       |   scope=payment:write
       |                             |
       |<-- access_token ------------|

No user involved.
Service authenticates with its own credentials.
Used for: scheduled jobs, internal service-to-service calls,
          background workers, webhooks
```

**3. Implicit Flow (deprecated — do not use):**
```
Token returned directly in URL fragment
→ Token exposed in browser history
→ Replaced by Authorization Code + PKCE
→ Never use in new applications
```

**4. Resource Owner Password Flow (deprecated — avoid):**
```
User gives username/password directly to client app
Client sends credentials to auth server
→ Defeats purpose of OAuth2 (client sees password)
→ Only for legacy migration
→ Never use in new systems
```

---

## OpenID Connect (OIDC) — Authentication on Top of OAuth2

**OAuth2 is an authorization framework — it was not designed for authentication.**

OAuth2 gives you an access token that says "this app can access these resources." It does not tell you WHO the user is.

**OpenID Connect (OIDC)** adds an identity layer on top of OAuth2:

```
OAuth2:  gives you access_token (authorization)
OIDC:    gives you access_token + id_token (authorization + authentication)

id_token is a JWT containing:
{
  "sub":    "google-user-id-12345",    ← user's unique ID at Google
  "email":  "vidhan@gmail.com",
  "name":   "Vidhan",
  "picture": "https://...",
  "iss":    "https://accounts.google.com",
  "aud":    "shopsphere-client-id",
  "iat":    1712566200,
  "exp":    1712567100
}

ShopSphere:
  Receives id_token
  Verifies signature against Google's public keys
  Extracts email and sub
  Looks up or creates local user account
  Issues ShopSphere-specific JWT
  User is now authenticated
```

**The userinfo endpoint:**
```
GET https://openid.googleapis.com/v1/userinfo
Authorization: Bearer <access_token>

→ Returns user profile
   { "sub": "...", "email": "...", "name": "...", "picture": "..." }

Alternative to id_token for fetching user profile
```

---

## Scopes — Limiting What Tokens Can Do

Scopes define the permissions a token grants — implementing least privilege for OAuth2:

```
Standard OIDC scopes:
  openid      ← required for OIDC — enables id_token
  email       ← access to email address
  profile     ← access to name, picture, locale
  address     ← access to postal address
  phone       ← access to phone number

ShopSphere custom scopes:
  orders:read      ← read orders
  orders:write     ← create/modify orders
  products:read    ← read products
  products:write   ← manage products
  admin            ← administrative access
  
Token with scope "orders:read":
  Can call GET /orders
  CANNOT call POST /orders (needs orders:write)
  CANNOT call /admin/* (needs admin scope)
```

```java
// Resource server validates scope
@GetMapping("/orders")
@PreAuthorize("hasAuthority('SCOPE_orders:read')")
public List<Order> getOrders() { ... }

@PostMapping("/orders")
@PreAuthorize("hasAuthority('SCOPE_orders:write')")
public Order createOrder(@RequestBody CreateOrderRequest request) { ... }
```

---

## Putting It Together — ShopSphere Auth Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Auth Service                        │
│   (Spring Authorization Server / Keycloak)           │
│                                                      │
│   POST /oauth2/token     ← issues tokens             │
│   POST /oauth2/refresh   ← refreshes tokens          │
│   POST /oauth2/logout    ← revokes refresh tokens    │
│   GET  /.well-known/     ← OIDC discovery endpoint   │
│         jwks.json        ← public keys               │
└──────────────────────────┬──────────────────────────┘
                           │ JWT public keys (cached)
                           ▼
┌─────────────────────────────────────────────────────┐
│                  API Gateway                         │
│   Validates JWT on every request locally             │
│   Extracts userId, roles, scopes from claims         │
│   Forwards X-User-Id and X-User-Roles headers        │
│   Rejects 401 if invalid, 403 if insufficient role   │
└──────────────────────────┬──────────────────────────┘
                           │ trusted internal headers
                           ▼
┌──────────────┐  ┌───────────────┐  ┌────────────────┐
│ Order Service│  │Product Service│  │ User Service   │
│              │  │               │  │                │
│ Reads userId │  │ Reads userId  │  │ Reads userId   │
│ from header  │  │ from header   │  │ from header    │
│              │  │               │  │                │
│ @PreAuthorize│  │ @PreAuthorize │  │ @PreAuthorize  │
│ ownership    │  │ seller check  │  │ self-only check│
│ checks       │  │               │  │                │
└──────────────┘  └───────────────┘  └────────────────┘
```

```java
// Complete login flow in Auth Service
@RestController
@RequestMapping("/auth")
public class AuthController {

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @RequestBody LoginRequest request,
            HttpServletResponse response) {

        // 1. Verify credentials
        User user = userService.findByEmail(request.getEmail());
        if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
            throw new InvalidCredentialsException();
        }

        // 2. Check MFA if enabled
        if (user.isMfaEnabled()) {
            if (!totpService.verify(user.getMfaSecret(), request.getOtpCode())) {
                throw new InvalidOtpException();
            }
        }

        // 3. Generate access token (15 minutes)
        String accessToken = jwtService.generateAccessToken(
            user.getId(),
            user.getEmail(),
            user.getRoles(),
            Duration.ofMinutes(15)
        );

        // 4. Generate refresh token (30 days) and store in DB
        String refreshToken = refreshTokenService.generate(user.getId());

        // 5. Set refresh token as httpOnly cookie
        ResponseCookie cookie = ResponseCookie.from("refresh_token", refreshToken)
            .httpOnly(true)           // not accessible to JavaScript
            .secure(true)             // HTTPS only
            .sameSite("Strict")       // no cross-site requests
            .maxAge(Duration.ofDays(30))
            .path("/auth")            // only sent to /auth endpoints
            .build();
        response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());

        // 6. Return access token in body
        return ResponseEntity.ok(new LoginResponse(accessToken, user.getId()));
    }

    @PostMapping("/refresh")
    public ResponseEntity<RefreshResponse> refresh(
            @CookieValue("refresh_token") String refreshToken,
            HttpServletResponse response) {

        // Verify refresh token exists and is not revoked
        RefreshTokenRecord record = refreshTokenService.validate(refreshToken);

        // Rotate — invalidate old token, issue new one
        refreshTokenService.invalidate(refreshToken);
        String newRefreshToken = refreshTokenService.generate(record.getUserId());
        String newAccessToken = jwtService.generateAccessToken(
            record.getUserId(), record.getEmail(), record.getRoles(),
            Duration.ofMinutes(15)
        );

        // Set new refresh token cookie
        setRefreshTokenCookie(response, newRefreshToken);

        return ResponseEntity.ok(new RefreshResponse(newAccessToken));
    }
}
```

---

## Interview Q&A

**Q: What is a JWT and how does it work?**
A JWT is a self-contained token consisting of three Base64URL-encoded parts — header, payload, and signature — separated by dots. The header declares the signing algorithm. The payload carries claims about the subject — userId, roles, expiry. The signature is created by the auth service using its private key and verified by any service using the corresponding public key. This allows stateless authentication — any service can verify a JWT locally without calling the auth service, by checking the cryptographic signature and expiry claims.

**Q: What is the difference between access tokens and refresh tokens?**
Access tokens are short-lived — typically 15 minutes — stateless, and used for every API request. They are verified locally via JWT signature without any database lookup. Refresh tokens are long-lived — typically 7 to 30 days — stored in a database, and used only to obtain new access tokens when the current one expires. Storing refresh tokens in the database enables revocation — logging out invalidates the refresh token immediately, even though the access token remains valid until its natural expiry.

**Q: What is OAuth2 and what problem does it solve?**
OAuth2 is an authorization framework that solves delegated access — allowing a third-party application to access a user's resources without the user sharing their password. Instead of giving ShopSphere your Google password, you authenticate directly with Google, and Google issues ShopSphere a scoped access token. ShopSphere can only do what the token's scopes permit, the user can revoke access at any time, and their Google password is never exposed. OAuth2 is the framework; the tokens are often JWTs.

**Q: What is PKCE and why is it required for mobile apps?**
PKCE — Proof Key for Code Exchange — prevents authorization code interception attacks. The client generates a random code verifier, sends a hash of it as the code challenge in the authorization request, and presents the original verifier when exchanging the code for tokens. The auth server verifies the hash matches. On mobile, the redirect URI can be intercepted by a malicious app registered for the same scheme — without PKCE, the attacker could exchange the intercepted code for tokens. With PKCE, they cannot because they do not have the code verifier.

**Q: How would you implement token revocation without sacrificing statelessness?**
Pure JWT revocation requires a token blacklist — a Redis set of revoked JTI values checked on every request, which reintroduces statefulness. The pragmatic approach is to accept that access tokens cannot be truly revoked before expiry — keep them very short-lived so the window of exposure is small. Refresh tokens are stored in the database and can be revoked instantly. On logout, the refresh token is deleted — the user cannot obtain new access tokens, and the current access token expires within minutes. For high-security operations like password changes or suspicious activity detection, issue new JWT signing keys on a rotation — all tokens signed with old keys become invalid.

---

Say **"next"** when ready for the final topic of Stage 2 — Topic 11: API Gateway.
