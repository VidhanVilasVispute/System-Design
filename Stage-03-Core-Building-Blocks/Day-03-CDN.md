# Stage 3 — Topic 3: CDN (Content Delivery Network)

## Theory

Every time a user in Mumbai requests an image stored on a server in Virginia, that request travels roughly 13,000 kilometres. At the speed of light through fibre optic cables, that is about 150ms of round-trip latency — before any processing happens. For a product page that loads 30 images, that is 30 × 150ms = 4.5 seconds of pure distance penalty.

A **CDN solves the distance problem** by caching your content at servers distributed geographically close to your users — called **edge nodes** or **Points of Presence (PoPs)**. Instead of travelling to Virginia, that user in Mumbai hits a CDN edge node in Mumbai. Latency drops from 150ms to 5ms.

**The core insight:**
```
Most content served on the internet does not change frequently.
A product image, a CSS file, a JavaScript bundle — served
millions of times, unchanged, to users all over the world.

There is no reason for all those users to travel to one origin server.
Serve them from nearby — once the content is cached there.
```

---

## How a CDN Works — The Full Flow

### First Request — Cache Miss

```
User in Mumbai
  │
  ▼
DNS lookup: cdn.shopsphere.com
  → Anycast routing returns Mumbai PoP IP (nearest edge)
  │
  ▼
CDN Edge Node — Mumbai
  Check cache: /images/product/p-456.webp → MISS
  │
  ▼
CDN fetches from Origin (Singapore):
  GET https://origin.shopsphere.com/images/product/p-456.webp
  │
  ▼
Origin returns:
  HTTP 200
  Cache-Control: public, max-age=86400   ← tells CDN to cache for 24 hours
  ETag: "abc123"
  Content-Type: image/webp
  [image bytes]
  │
  ▼
CDN Edge Node — Mumbai:
  Stores image in local cache with 24-hour TTL
  Returns image to user
  │
  ▼
User receives image in ~8ms  ← instead of ~150ms
```

### Subsequent Requests — Cache Hit

```
Any user in Mumbai requesting same image:
  CDN Edge Node — Mumbai
    Check cache: /images/product/p-456.webp → HIT
    Return cached image immediately
    Origin server never contacted
    Latency: ~5ms
    Origin bandwidth used: 0
```

### CDN Miss Ratio Economics

```
Product image requested 1,000,000 times/day globally
CDN cache hit ratio: 95%

Requests served from CDN edge: 950,000  ← fast, cheap, no origin load
Requests hitting origin:         50,000  ← origin only handles 5%

Without CDN:
  Origin handles 1,000,000 requests/day
  Bandwidth cost: 1M × 200KB = 200GB/day outbound
  
With CDN (95% hit ratio):
  Origin handles 50,000 requests/day
  Bandwidth cost: 50K × 200KB = 10GB/day to CDN origin
  CDN handles 190GB/day internally — much cheaper per GB
```

---

## CDN Architecture — PoPs and Anycast

### Points of Presence (PoPs)

A CDN provider like Cloudflare, AWS CloudFront, or Akamai operates hundreds of PoPs globally — data centres with edge servers in major cities:

```
Cloudflare PoPs (subset):
  Asia:    Mumbai, Singapore, Tokyo, Seoul, Sydney, Bangkok
  Europe:  London, Paris, Frankfurt, Amsterdam, Madrid
  Americas: New York, Los Angeles, São Paulo, Toronto
  Africa:  Johannesburg, Cairo
  
300+ PoPs globally = most users within 20ms of an edge node
```

### Anycast Routing — How Users Hit the Nearest PoP

```
Traditional Unicast:
  api.shopsphere.com → one specific IP → one specific server
  All users go to that one server regardless of distance

CDN Anycast:
  cdn.shopsphere.com → same IP announced from 300+ locations
  Internet routing automatically sends each user to the
  nearest announcing location (BGP routing)
  
User in Mumbai → Mumbai PoP (same IP, geographically nearest)
User in London → London PoP (same IP, geographically nearest)
User in New York → New York PoP (same IP, geographically nearest)

One IP, hundreds of physical destinations — user always routed
to nearest without any application-level logic
```

---

## Pull CDN vs Push CDN

Two fundamentally different models for getting content onto the CDN:

### Pull CDN

```
Content lives on your origin server.
CDN fetches it on demand (on first cache miss).
You do nothing special — just set correct Cache-Control headers.

Flow:
  User requests → CDN edge → MISS → CDN fetches from origin
  → caches it → serves user → all future requests from cache

Pros:
  Zero upfront work — just configure DNS and headers
  Only popular content gets cached — no wasted edge storage
  Origin remains source of truth — no sync needed

Cons:
  First user after cache expiry pays the origin round-trip penalty
  Origin must be publicly accessible (or CDN must have access)
  Cache warm-up after deployment → first N users see slower response

Best for:
  Images, CSS, JavaScript, API responses
  Content that is requested organically
  Most web applications — this is the default CDN model
```

### Push CDN

```
You explicitly upload content to CDN edge nodes before users request it.
Content is pushed to all PoPs proactively.

Flow:
  You upload new product image → push to all 300 PoPs
  First user request → cache HIT everywhere immediately

Pros:
  No cache miss ever for pushed content
  No origin dependency after push — edge is self-sufficient
  Predictable performance for all users

Cons:
  You must manage what is pushed and when
  Wasted storage for unpopular content pushed everywhere
  Sync complexity — keeping edge copies up to date
  More expensive — you pay for storage at all PoPs

Best for:
  Software downloads — large files, must be fast for all users
  Game patches, OS updates, video files
  Content with known distribution requirements upfront
```

---

## Cache Control — Telling CDN What to Cache and For How Long

The `Cache-Control` header is your primary tool for controlling CDN behaviour:

```
Cache-Control: public, max-age=31536000, immutable
  public      → can be cached by CDN (and browsers)
  max-age     → cache for 1 year (in seconds)
  immutable   → content will never change — don't revalidate

Cache-Control: public, max-age=300
  Cache for 5 minutes — then CDN revalidates with origin

Cache-Control: private, no-store
  private   → only browser can cache, NOT CDN
  no-store  → don't cache at all (sensitive data)

Cache-Control: no-cache
  Must revalidate with origin before serving from cache
  (confusingly named — it does cache, just always validates)
  
ETag: "abc123"
  Fingerprint of the content
  CDN sends: If-None-Match: "abc123" on revalidation
  Origin responds: 304 Not Modified (if unchanged) → CDN serves cache
                   200 OK with new content (if changed) → CDN updates cache
```

**ShopSphere cache strategy by content type:**

```
Product images (immutable with hash in URL):
  Cache-Control: public, max-age=31536000, immutable
  URL: /images/p-456-a3f9b2c1.webp  ← hash in filename
  Cache forever — URL changes when image changes

Product data API (changes occasionally):
  Cache-Control: public, max-age=300
  Cache for 5 minutes — stale data is acceptable
  
Order data API (user-specific, must be fresh):
  Cache-Control: private, no-store
  Never cache — each user sees their own orders

Search results (semi-static):
  Cache-Control: public, max-age=60
  Cache for 1 minute — acceptable staleness for search

JavaScript/CSS bundles (hash in filename):
  Cache-Control: public, max-age=31536000, immutable
  /static/main.a3b4c5d6.js ← content hash in filename
  Cache forever — new deploy = new hash = new URL
```

---

## Cache Invalidation — The Hard Problem

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

When content changes before its TTL expires, you need to invalidate the cached version:

### TTL-Based Expiry

```
Simplest approach — just wait for TTL:
  Product image cached with max-age=86400 (24 hours)
  Image updated at noon
  Users see old image until midnight (up to 24 hours stale)

Acceptable for: slow-changing content
Not acceptable for: price changes, inventory updates, urgent corrections
```

### Purge API — Immediate Invalidation

```
CDN providers offer a purge API:
  DELETE https://api.cloudflare.com/zones/{zoneId}/purge_cache
  Body: { "files": ["https://cdn.shopsphere.com/images/p-456.webp"] }

After product image update:
  1. Upload new image to origin
  2. Call CDN purge API for that URL
  3. CDN immediately marks that URL as expired at all PoPs
  4. Next request fetches fresh content from origin

Wildcard purge:
  { "prefixes": ["cdn.shopsphere.com/images/"] }
  Purge all images at once — useful after bulk update

Cost: purge APIs are often rate-limited and priced
     Use sparingly — not a solution to poor TTL strategy
```

### Cache-Busting — The Better Solution

```
Instead of invalidating — change the URL:
  
  Old:  /static/app.js          → cached at CDN with max-age=1year
  New:  /static/app.a3b4c5d6.js → brand new URL → no cache exists yet

  Content hash in filename:
    URL contains hash of file contents
    Same content → same hash → same URL (cache hit)
    Changed content → new hash → new URL → cache miss → fresh fetch

  This is what Webpack, Vite, and build tools do automatically.
  Every build produces filenames with content hashes.
  
Benefits:
  max-age can be set to 1 year with immutable
  No purge API calls needed
  Old files stay cached — old browsers loading old HTML still work
  Zero race conditions — old and new versions coexist in CDN
```

---

## What CDNs Cache — Beyond Static Files

Modern CDNs can cache far more than just images and CSS:

### API Response Caching

```java
// Cache product API responses at CDN for 5 minutes
@GetMapping("/products/{productId}")
public ResponseEntity<ProductResponse> getProduct(@PathVariable String productId) {
    ProductResponse product = productService.getProduct(productId);
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl
            .maxAge(300, TimeUnit.SECONDS)
            .public_()                    // CDN can cache
            .mustRevalidate())            // revalidate after expiry
        .eTag(product.getVersion())       // ETag for conditional requests
        .body(product);
}

// Never cache user-specific responses
@GetMapping("/orders")
public ResponseEntity<List<Order>> getMyOrders(
        @AuthenticationPrincipal UserDetails user) {
    return ResponseEntity.ok()
        .cacheControl(CacheControl.noStore())   // never cache
        .body(orderService.getOrders(user.getId()));
}
```

### Vary Header — Different Caches for Different Clients

```
Vary: Accept-Encoding
  Cache separate versions for gzip and non-gzip clients
  Gzip client gets compressed version from CDN cache
  Non-gzip client gets uncompressed version

Vary: Accept-Language
  Cache separate versions for different language preferences
  English user gets English version from nearest PoP
  Hindi user gets Hindi version from same PoP

Vary: Authorization
  NEVER do this — caches one user's response for another
  For authenticated content, use Cache-Control: private
```

---

## CDN for Video — Adaptive Bitrate Streaming

Video is the highest-bandwidth CDN use case. Modern video delivery uses **Adaptive Bitrate Streaming (ABR)**:

```
Video encoded at multiple quality levels:
  1080p:  8 Mbps
  720p:   4 Mbps
  480p:   1.5 Mbps
  360p:   0.8 Mbps
  240p:   0.4 Mbps

Video split into small segments (2-10 seconds each):
  segment_001_1080p.ts
  segment_001_720p.ts
  segment_001_480p.ts
  ...

Player downloads a manifest (HLS .m3u8 or DASH .mpd):
  Lists all segments and their URLs

Player logic:
  Measure download speed → pick appropriate quality
  Network slows down → switch to lower quality mid-stream
  Network improves → switch to higher quality
  All segments served from CDN edge — extremely fast

ShopSphere video review feature:
  Seller uploads product demo video
  Transcoding service creates 5 quality levels
  All segments pushed to CDN
  Buyer streams appropriate quality based on connection speed
```

---

## CDN Security Features

Modern CDNs provide significant security capabilities:

```
DDoS Protection:
  CDN absorbs volumetric attacks at the edge
  Cloudflare handles 100+ Gbps attacks
  Traffic scrubbed before reaching origin

Web Application Firewall (WAF):
  Block SQL injection, XSS, CSRF at CDN edge
  OWASP rule sets applied to all requests
  Malicious requests never reach origin

Bot Protection:
  Detect and block scrapers, credential stuffers
  Challenge suspicious traffic with CAPTCHA
  Rate limit by IP, ASN, or behavioural patterns

Hotlink Protection:
  Prevent other sites from embedding your images
  Check Referer header — reject if not your domain
  Prevents bandwidth theft

Signed URLs:
  Time-limited, cryptographically signed URLs for private content
  cdn.shopsphere.com/invoices/inv-789.pdf?
    expires=1712570000&signature=abc123
  
  CDN verifies signature — rejects expired or tampered URLs
  Private content served from CDN without exposing it publicly
```

---

## Real-World Example — ShopSphere CDN Architecture

```
ShopSphere uses AWS CloudFront as its CDN:

Distribution 1 — Static Assets (immutable):
  Origin:         S3 bucket (shopsphere-static)
  Cache-Control:  public, max-age=31536000, immutable
  Domains:        static.shopsphere.com
  Content:        JS, CSS, fonts — hash in filename
  Invalidation:   Never needed (hash-based URLs)

Distribution 2 — Product Images:
  Origin:         S3 bucket (shopsphere-images)
  Cache-Control:  public, max-age=86400
  Domains:        images.shopsphere.com
  Content:        Product photos, thumbnails, banners
  Invalidation:   Purge API on product image update

Distribution 3 — API Responses:
  Origin:         ALB → API Gateway
  Cache-Control:  public, max-age=300  (product endpoints)
                  private, no-store    (order/user endpoints)
  Domains:        api.shopsphere.com
  Content:        Public API responses (product listings, search)

Distribution 4 — Video Content:
  Origin:         S3 bucket (shopsphere-videos)
  Cache-Control:  public, max-age=3600
  Domains:        video.shopsphere.com
  Content:        HLS segments, manifests
  Edge Locations: All CloudFront PoPs globally
```

**Impact numbers:**
```
Before CDN:
  Average product page load: 3.2 seconds (Mumbai → Singapore)
  Origin bandwidth: 800GB/day
  Origin request load: 2M requests/day

After CDN:
  Average product page load: 0.4 seconds (Mumbai → Mumbai PoP)
  Origin bandwidth: 40GB/day  (95% cache hit ratio)
  Origin request load: 100K requests/day
  
80× reduction in origin load
8× improvement in page load time
```

---

## Interview Q&A

**Q: What is a CDN and what problem does it solve?**
A CDN is a geographically distributed network of edge servers that cache content close to users. It solves the distance problem — a user in Mumbai requesting content from a server in Virginia faces ~150ms of round-trip latency just from physical distance. With a CDN, that same user hits an edge node in Mumbai in ~5ms. Beyond latency, CDNs dramatically reduce origin server load — a 95% cache hit ratio means the origin handles only 5% of total requests, reducing bandwidth costs and preventing the origin from being overwhelmed by traffic spikes.

**Q: What is the difference between a pull CDN and a push CDN?**
A pull CDN fetches content from your origin on the first cache miss and caches it at the edge for subsequent requests. You only need to set correct Cache-Control headers — the CDN handles everything else. Content is cached organically based on demand. A push CDN requires you to explicitly upload content to all edge nodes before users request it. This guarantees no cache misses but wastes storage for unpopular content and requires managing the push workflow. Pull CDNs are the default for web applications. Push CDNs suit large file distribution — software downloads, game patches — where you need guaranteed performance from the first request for all users.

**Q: What is cache busting and why is it better than calling the CDN purge API?**
Cache busting embeds a content hash in the filename — `app.a3b4c5d6.js`. When content changes, the hash changes, creating a new URL that has never been cached. The CDN serves the new file immediately on first request. The purge API invalidates cached content at a specific URL across all edge nodes — it works but has operational downsides: it is rate-limited, costs money per purge, can take seconds to propagate globally, and creates a race condition where some users get old content while invalidation propagates. Cache busting with content-hashed URLs allows setting `max-age=31536000, immutable` — cache forever with zero invalidation overhead. Build tools like Webpack and Vite generate these automatically.

**Q: How would you decide which API responses to cache at the CDN?**
Cache responses that are public — not user-specific — and where brief staleness is acceptable. Product listings, product detail pages, search results, and category pages are good candidates — they are the same for all users and a 1 to 5 minute stale window is fine. Set `Cache-Control: public, max-age=300`. Never cache authenticated responses — orders, user profiles, cart contents — these are user-specific and must be fresh. Set `Cache-Control: private, no-store`. The rule of thumb is — if the same URL returns different content for different users, it must not be CDN-cached.

**Q: What is a signed URL and when would you use it in ShopSphere?**
A signed URL is a time-limited URL with a cryptographic signature that the CDN verifies before serving content. It enables serving private content through the CDN — which normally only caches public content. In ShopSphere, signed URLs are used for invoice PDFs, order receipts, and seller analytics reports — content that should be fast to download via CDN but must not be publicly accessible. The URL contains an expiry timestamp and a signature generated with a secret key. If the URL is shared, it expires — the CDN rejects requests with expired or invalid signatures without ever reaching the origin.

---

Say **"next"** when ready for Topic 4 — Caching.
