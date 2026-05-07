# Stage 3 — Topic 2: Forward Proxy vs Reverse Proxy

## Theory

Proxy is one of those words that gets used loosely in engineering conversations. When someone says "we use a proxy," you need to ask — which direction? A forward proxy and a reverse proxy sit on opposite sides of the client-server relationship, serve completely different purposes, and are confused constantly in interviews.

**The one-line distinction:**
- **Forward proxy** — sits in front of clients, hides the client from the server
- **Reverse proxy** — sits in front of servers, hides the server from the client

```
Forward Proxy:
  Client → [Forward Proxy] → Internet → Server
  Server sees proxy IP, not client IP
  Client knows about the proxy
  Server does not know about the proxy

Reverse Proxy:
  Client → Internet → [Reverse Proxy] → Server
  Client sees proxy IP, not server IP
  Server knows about the proxy
  Client does not know about the proxy
```

The word "reverse" simply means it is the mirror image of a forward proxy — everything flipped.

---

## Forward Proxy — Deep Dive

### What It Is

A forward proxy is an intermediary that clients use to make requests on their behalf. The client is configured to send all its traffic through the proxy instead of directly to the destination.

```
Corporate Network:

Employee laptop → [Corporate Forward Proxy] → Internet

Without proxy:
  Laptop → google.com directly

With proxy:
  Laptop → proxy.company.com → google.com
  google.com sees request from proxy.company.com
  google.com never sees the employee's laptop IP
```

### Use Cases

**1. Privacy and Anonymity:**
```
Home user → [VPN / Forward Proxy] → Website
  Website sees proxy's IP in Singapore
  User's real IP in India is hidden
  
This is what VPNs are — a forward proxy with encryption
```

**2. Content Filtering — Corporate and School Networks:**
```
All outbound traffic → Forward Proxy → Internet
  Proxy checks destination against blocklist:
    facebook.com   → BLOCKED
    youtube.com    → BLOCKED (during work hours)
    api.stripe.com → ALLOWED
    
Employees cannot bypass — all traffic must go through proxy
Proxy logs all requests — complete audit trail
```

**3. Caching Outbound Requests:**
```
100 employees all open the same news website:
  Without proxy: 100 requests to news.com
  With caching proxy: 1 request to news.com, 99 served from cache
  
Saves bandwidth on corporate network egress
```

**4. Bypassing Geographic Restrictions:**
```
User in India → [Proxy in US] → US-only content
Content server sees US IP → serves content
User bypasses geo-block
```

**5. Security Scanning:**
```
All outbound requests pass through proxy:
  Proxy inspects for data exfiltration
  Scans for malware in downloads
  Prevents employees from uploading sensitive data to external services
```

### How a Forward Proxy Works Technically

```
HTTP (non-tunnelled):
  Client sends full URL to proxy:
    GET http://api.example.com/products HTTP/1.1
    Host: api.example.com
    
  Proxy connects to api.example.com and forwards
  Proxy can read and modify the request

HTTPS (CONNECT tunnel):
  Client sends CONNECT method:
    CONNECT api.example.com:443 HTTP/1.1
    Host: api.example.com:443
    
  Proxy opens TCP tunnel to api.example.com:443
    HTTP/1.1 200 Connection established
    
  Client sends encrypted HTTPS through the tunnel
  Proxy cannot read the content — it is encrypted
  Proxy just passes bytes through
  
Corporate proxies doing SSL inspection:
  Proxy generates a fake certificate signed by corporate CA
  Installs corporate CA on all employee machines
  Now proxy can decrypt and inspect HTTPS
  (This is why you should never use corporate laptops for personal things)
```

---

## Reverse Proxy — Deep Dive

### What It Is

A reverse proxy sits in front of your servers and intercepts all incoming requests before they reach your backend. Clients think they are talking to your server — they are actually talking to the proxy.

```
Client requests api.shopsphere.com:
  DNS: api.shopsphere.com → 54.12.45.100 (reverse proxy IP)
  
  Client → [Reverse Proxy at 54.12.45.100] → Backend servers at 10.0.x.x
  
  Backend servers are in a private network
  Client only ever sees the proxy's IP
  Backend IPs are never exposed publicly
```

### Use Cases

**1. SSL Termination:**
```
Client ──HTTPS──► Reverse Proxy ──HTTP──► Backend servers
  
  One SSL certificate on the proxy
  Proxy decrypts once — backends never deal with SSL
  Centralised certificate renewal
  Lower CPU usage on backend servers
```

**2. Load Balancing:**
```
Reverse Proxy distributes requests across multiple backends:
  Request 1 → Server A
  Request 2 → Server B
  Request 3 → Server C
  
Most reverse proxies have built-in load balancing.
This is why Nginx is often used as both a reverse proxy and load balancer.
```

**3. Caching:**
```
GET /products/p-456 (first request):
  Proxy → Backend → Product Service → DB
  Proxy caches response: { product data }

GET /products/p-456 (subsequent requests):
  Proxy → cache hit → returns cached response
  Backend never touched
  
Dramatically reduces backend load for read-heavy endpoints
```

**4. Compression:**
```
Backend returns uncompressed JSON: 50KB
Reverse proxy compresses with gzip: 8KB
Client receives 8KB

Backend does not need to implement compression
Proxy handles it for all services
```

**5. DDoS Protection and Rate Limiting:**
```
Attacker sends 1,000,000 requests/second:
  Reverse proxy absorbs the flood
  Rate limits to 1000 req/s per IP
  Backend never sees the flood
  
Cloudflare, AWS CloudFront — all reverse proxies doing DDoS protection
```

**6. Hiding Backend Architecture:**
```
Client sees:
  api.shopsphere.com/orders   → one server
  api.shopsphere.com/products → same server
  
Reality (proxy routes internally):
  /orders/**   → order-service:8081
  /products/** → product-service:8082
  /search/**   → search-service:8084
  
Client has no idea there are 22 separate services
Architecture can change internally without changing client-facing URLs
```

**7. A/B Testing and Canary Deployments:**
```
Reverse proxy routes based on header or cookie:
  Cookie: beta=true  → new version (10% of users)
  Default            → stable version (90% of users)
  
No client changes needed — purely proxy routing logic
```

---

## Nginx — The Most Common Reverse Proxy

Nginx is the most widely used reverse proxy and web server. Understanding its config is practical knowledge:

```nginx
# Nginx as reverse proxy for ShopSphere

# Upstream definitions — backend server pools
upstream order_service {
    least_conn;                          # algorithm: least connections
    server order-1.internal:8081 weight=3;
    server order-2.internal:8081 weight=3;
    server order-3.internal:8081 weight=2;  # new server, lower weight
    keepalive 32;                        # keep 32 connections open
}

upstream product_service {
    server product-1.internal:8082;
    server product-2.internal:8082;
    server product-3.internal:8082;
}

# Rate limiting zone — track by IP
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;

server {
    listen 443 ssl http2;
    server_name api.shopsphere.com;

    # SSL termination
    ssl_certificate     /etc/ssl/shopsphere.crt;
    ssl_certificate_key /etc/ssl/shopsphere.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Gzip compression
    gzip on;
    gzip_types application/json text/plain;
    gzip_min_length 1024;

    # Security headers
    add_header X-Frame-Options "DENY";
    add_header X-Content-Type-Options "nosniff";
    add_header Strict-Transport-Security "max-age=31536000";

    # Rate limiting
    limit_req zone=api_limit burst=200 nodelay;

    # Path-based routing to different services
    location /api/v1/orders/ {
        limit_req zone=api_limit burst=50;

        # Forward to order service pool
        proxy_pass http://order_service;

        # Pass original client IP to backend
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header Host            $http_host;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout    30s;
        proxy_send_timeout    30s;
    }

    location /api/v1/products/ {
        # Cache product responses
        proxy_cache product_cache;
        proxy_cache_valid 200 5m;          # cache 200 responses for 5 minutes
        proxy_cache_key $uri$is_args$args; # cache key includes query params

        proxy_pass http://product_service;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        add_header X-Cache-Status $upstream_cache_status; # HIT or MISS in response
    }

    location /api/v1/search/ {
        proxy_pass http://search_service;
        proxy_read_timeout 10s;             # search should be fast
    }

    # Health check endpoint — not proxied, served by Nginx directly
    location /health {
        access_log off;
        return 200 '{"status":"UP"}';
        add_header Content-Type application/json;
    }

    # Redirect HTTP to HTTPS
    error_page 497 https://$host$request_uri;
}

# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name api.shopsphere.com;
    return 301 https://$host$request_uri;
}
```

---

## X-Forwarded-For — The Important Header

When a reverse proxy forwards a request, the backend server sees the proxy's IP as the client IP — not the real client IP. `X-Forwarded-For` solves this:

```
Client IP: 203.45.67.89
  ↓
Reverse Proxy IP: 10.0.0.1
  ↓ adds header:
  X-Forwarded-For: 203.45.67.89
  ↓
Backend Server
  reads X-Forwarded-For: 203.45.67.89
  knows real client IP for:
    - Rate limiting per client IP
    - Geo-based routing
    - Fraud detection
    - Audit logging

Chained proxies:
  X-Forwarded-For: 203.45.67.89, 10.0.0.1, 10.0.0.5
  Leftmost IP is the original client (most trusted in controlled environments)
  
Security note:
  Clients can spoof this header if your proxy doesn't strip it first
  Always strip incoming X-Forwarded-For and let your proxy set it
```

---

## Forward Proxy vs Reverse Proxy — Side by Side

| Dimension | Forward Proxy | Reverse Proxy |
|---|---|---|
| Sits in front of | Clients | Servers |
| Who configures it | Client / client's network admin | Server / server's operator |
| Who knows about it | Client knows, server doesn't | Server knows, client doesn't |
| Hides | Client identity from server | Server identity from client |
| Primary use cases | Privacy, filtering, caching outbound | Load balancing, SSL termination, caching, security |
| Examples | VPN, Squid, corporate proxy | Nginx, HAProxy, AWS ALB, Cloudflare |
| Typical location | Client network edge | Server network edge |

---

## Combining Both — The Full Picture

In a large system you often have both:

```
Employee laptop
    │
    ▼
[Corporate Forward Proxy]   ← hides employee from internet
    │
    ▼
Internet
    │
    ▼
[Cloudflare — Reverse Proxy]  ← hides ShopSphere servers from internet
    │  DDoS protection
    │  CDN caching
    │  SSL termination
    ▼
[Nginx — Reverse Proxy]       ← internal routing
    │  Load balancing
    │  Compression
    │  Auth forwarding
    ▼
ShopSphere API Gateway
    │
    ▼
Microservices
```

The employee's request passes through a forward proxy on the way out and a reverse proxy on the way in — without either side knowing about the other.

---

## Real-World Example — ShopSphere Proxy Architecture

```
ShopSphere production request flow:

1. User in Mumbai opens ShopSphere mobile app
   GET https://api.shopsphere.com/api/v1/products?q=shoes

2. DNS resolves api.shopsphere.com → Cloudflare edge IP in Mumbai

3. Cloudflare (Reverse Proxy — Edge):
   - Checks for DDoS pattern — clean, allows through
   - Checks CDN cache — MISS (API response, not static)
   - Terminates SSL — decrypts HTTPS
   - Forwards to origin: GET http://api.shopsphere.com/api/v1/products?q=shoes
     X-Forwarded-For: 203.45.67.89 (user's real IP)
     CF-Connecting-IP: 203.45.67.89

4. AWS ALB (Reverse Proxy — Load Balancer):
   - Receives from Cloudflare
   - Health checks all API Gateway pods — all healthy
   - Routes to api-gateway-pod-2 (least connections)
   - Preserves X-Forwarded-For header

5. API Gateway (Spring Cloud Gateway):
   - Validates JWT
   - Applies rate limiting against user IP
   - Routes /api/v1/products/** → product-service
   - Strips /api/v1 prefix → /products?q=shoes
   - Forwards to product-service ClusterIP

6. Kubernetes Service (Internal L4 Proxy):
   - Round-robins across product-service pods
   - Routes to product-pod-3

7. Product Service:
   - Reads X-User-Id from header (set by gateway)
   - Reads X-Forwarded-For for logging
   - Executes search logic
   - Returns product results

8. Response flows back through each layer in reverse
   Products rendered on user's screen in ~80ms
```

---

## Interview Q&A

**Q: What is the difference between a forward proxy and a reverse proxy?**
A forward proxy sits in front of clients — the client directs its traffic through it, and the server sees the proxy's IP instead of the client's. It is used for privacy, content filtering, and outbound caching. A reverse proxy sits in front of servers — clients send requests to it thinking it is the server, and it routes to backends the client cannot see. It is used for load balancing, SSL termination, caching, DDoS protection, and hiding backend architecture. The simplest way to remember it — forward proxy hides the client, reverse proxy hides the server.

**Q: Why does Nginx sit in front of a Spring Boot application in production?**
Nginx as a reverse proxy provides capabilities that application servers should not implement themselves — SSL termination centralises certificate management so Spring Boot services handle plain HTTP internally, load balancing distributes traffic across multiple instances, response caching reduces backend load for repeated requests, Gzip compression reduces bandwidth without application code changes, and rate limiting and security headers protect all services uniformly. It also handles static file serving far more efficiently than a JVM-based application, freeing the Spring Boot server for application logic only.

**Q: What is the X-Forwarded-For header and why does it matter?**
When a reverse proxy forwards a request, the backend sees the proxy's IP as the client — the real client IP is lost. X-Forwarded-For is a header the proxy adds containing the original client IP. Backend services read it for rate limiting per client, geo-based routing, fraud detection, and audit logging. In a chain of proxies the header accumulates IPs — the leftmost is the original client. The security consideration is that clients can spoof this header — your reverse proxy should strip any incoming X-Forwarded-For and set it fresh so backends can trust it.

**Q: How does a corporate forward proxy do SSL inspection if HTTPS is encrypted?**
When a client connects to an HTTPS site through a corporate forward proxy, the proxy intercepts the TLS handshake using a technique called SSL interception or man-in-the-middle. The corporate IT team installs a root CA certificate on all employee machines. The proxy generates a fake certificate for the destination site, signed by this corporate CA, which the browser trusts because the corporate CA is in its trusted store. The proxy decrypts the traffic, inspects it, then re-encrypts it to forward to the real destination. This is why employee browsers show the corporate CA in the certificate chain rather than the real site's CA.

**Q: What is the difference between Nginx and an API Gateway — can't Nginx do everything?**
Nginx as a reverse proxy handles infrastructure concerns excellently — SSL termination, load balancing, compression, static file serving, and basic routing. An API Gateway adds application-level intelligence — JWT validation and claims extraction, OAuth2 integration, per-user rate limiting with awareness of identity, service discovery integration, circuit breaking with fallbacks, request and response transformation, and WebSocket protocol handling. Nginx can approximate some of these with Lua scripting or the NGINX Plus commercial version, but dedicated API Gateways like Spring Cloud Gateway, Kong, or AWS API Gateway provide these natively with richer semantics. In practice, large systems use both — Nginx at the edge for raw performance, an API Gateway behind it for application logic.

---

Say **"next"** when ready for Topic 3 — CDN.
