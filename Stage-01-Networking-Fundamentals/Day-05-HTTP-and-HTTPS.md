# Topic 5: HTTP / HTTPS

## Theory

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol that powers the web. Every API call your microservices make, every request your frontend sends, every webhook that fires — it all runs over HTTP.

HTTP is built on top of TCP. It defines **how** requests and responses are structured and what they mean — not how they travel (that's TCP's job).

**HTTPS** is simply HTTP with **TLS (Transport Layer Security)** layered on top. It adds:
- **Encryption** — nobody can read the data in transit even if they intercept it
- **Authentication** — the client can verify it is talking to the real server, not an impostor
- **Integrity** — data cannot be tampered with in transit without detection

In 2024, there is no excuse to run plain HTTP in production. HTTPS is the baseline.

---

## Internals — HTTP Request Structure

Every HTTP request has three parts:

```
GET /api/products?category=shoes HTTP/1.1
Host: api.shopsphere.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json
Accept: application/json

{ "page": 1, "size": 20 }
```

- **Request Line** — method + path + HTTP version
- **Headers** — key-value metadata about the request
- **Body** — optional payload (present in POST, PUT, PATCH — not in GET, DELETE)

---

## Internals — HTTP Response Structure

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=300
X-Request-Id: abc-123

{ "products": [...], "total": 142 }
```

- **Status Line** — HTTP version + status code + reason phrase
- **Headers** — metadata about the response
- **Body** — the actual data returned

---

## HTTP Methods

| Method | Purpose | Idempotent | Has Body |
|---|---|---|---|
| GET | Fetch a resource | Yes | No |
| POST | Create a resource | No | Yes |
| PUT | Replace a resource entirely | Yes | Yes |
| PATCH | Partially update a resource | No | Yes |
| DELETE | Remove a resource | Yes | No |

**Idempotent** means calling it multiple times produces the same result. GET, PUT, DELETE are idempotent — POST is not. This matters for retry logic — you can safely retry a GET, but retrying a POST might create duplicate orders.

---

## HTTP Status Codes — The Ones That Matter

**2xx — Success**
- `200 OK` — standard success
- `201 Created` — resource was successfully created (use after POST)
- `204 No Content` — success but no body to return (use after DELETE)

**3xx — Redirection**
- `301 Moved Permanently` — resource has a new URL forever, update your bookmarks
- `302 Found` — temporary redirect
- `304 Not Modified` — cached version is still valid, no need to re-download

**4xx — Client Error** (the client did something wrong)
- `400 Bad Request` — malformed request, invalid input
- `401 Unauthorized` — not authenticated, you need to log in
- `403 Forbidden` — authenticated but not authorised, you don't have permission
- `404 Not Found` — resource doesn't exist
- `409 Conflict` — state conflict, e.g. trying to create a resource that already exists
- `422 Unprocessable Entity` — request is well-formed but semantically invalid
- `429 Too Many Requests` — rate limited

**5xx — Server Error** (the server did something wrong)
- `500 Internal Server Error` — generic server crash
- `502 Bad Gateway` — upstream service returned an invalid response
- `503 Service Unavailable` — server is down or overloaded
- `504 Gateway Timeout` — upstream service didn't respond in time

---

## HTTP Headers — The Important Ones

**Request Headers:**
- `Authorization` — carries credentials, e.g. `Bearer <JWT>`
- `Content-Type` — format of the request body, e.g. `application/json`
- `Accept` — format the client wants back, e.g. `application/json`
- `Cache-Control` — caching instructions from the client
- `X-Request-Id` — unique ID for tracing a request across services

**Response Headers:**
- `Content-Type` — format of the response body
- `Cache-Control` — how long the client/CDN should cache this response
- `Location` — used with 201/301/302 to point to the new resource URL
- `Set-Cookie` — instruct the client to store a cookie
- `X-RateLimit-Remaining` — how many requests the client has left in its window

---

## Internals — HTTPS and TLS Handshake

Before any HTTP data flows over HTTPS, a **TLS handshake** occurs:

```
Client                          Server
  |--- ClientHello ------------->|   "Here are the TLS versions and cipher suites I support"
  |<-- ServerHello + Certificate-|   "Let's use TLS 1.3, here's my certificate"
  |--- Verify Certificate ------>|   Client checks cert is signed by a trusted CA
  |--- Key Exchange ------------>|   Both sides derive a shared symmetric key
  |<== Encrypted HTTP begins ===>|   All HTTP traffic is now encrypted
```

Key points:
- The **certificate** contains the server's public key and is signed by a trusted **Certificate Authority (CA)** like Let's Encrypt, DigiCert
- The client verifies the certificate is valid, not expired, and signed by a CA it trusts
- After the handshake, all data is encrypted with a **symmetric key** — asymmetric crypto is too slow for bulk data
- **TLS 1.3** (current standard) reduced the handshake to 1 round-trip vs 2 in TLS 1.2

---

## HTTP Versions

**HTTP/1.1**
- One request at a time per TCP connection (head-of-line blocking)
- Keep-alive allows connection reuse across requests
- Still widely used

**HTTP/2**
- **Multiplexing** — multiple requests and responses in parallel over a single TCP connection
- **Header compression** (HPACK) — reduces overhead of repeated headers
- **Server push** — server can proactively send resources the client will need
- Significantly better performance than HTTP/1.1 for APIs with many small requests

**HTTP/3**
- Built on **QUIC** (UDP-based) instead of TCP
- Eliminates TCP head-of-line blocking entirely — a lost packet in one stream doesn't block others
- Faster connection setup — combines TLS and transport handshake in one round-trip
- Increasingly adopted by major platforms (Google, Cloudflare)

---

## Real-World Example — ShopSphere

**A complete request flow through ShopSphere:**

```
POST /api/orders HTTP/1.1
Host: api.shopsphere.com
Authorization: Bearer eyJhbGci...
Content-Type: application/json

{
  "userId": "u-123",
  "items": [{"productId": "p-456", "qty": 2}]
}
```

- API Gateway receives this, validates the JWT from the `Authorization` header
- Routes to Order Service — `POST /orders`
- Order Service creates the order, returns:

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/orders/o-789

{ "orderId": "o-789", "status": "PENDING" }
```

- If the Payment Service is down, Order Service returns `503 Service Unavailable`
- If the JWT is expired, API Gateway returns `401 Unauthorized` before it even reaches Order Service
- If the user tries to order a product they are banned from, `403 Forbidden`

---

## Interview Q&A

**Q: What is the difference between HTTP and HTTPS?**
HTTP transmits data in plain text — anyone who intercepts the traffic can read it. HTTPS wraps HTTP in TLS which encrypts the data, authenticates the server via certificates, and ensures integrity so data cannot be tampered with in transit.

**Q: What is the difference between 401 and 403?**
401 means the client is not authenticated — it has not proved who it is yet. 403 means the client is authenticated but not authorised — we know who you are, but you don't have permission to do this. A common interview trick question.

**Q: What does idempotent mean and why does it matter?**
An operation is idempotent if calling it multiple times produces the same result as calling it once. GET, PUT, and DELETE are idempotent — POST is not. This matters for retry logic — you can safely retry idempotent operations on failure without risk of duplicating state, like duplicate orders or double charges.

**Q: What is the difference between HTTP/1.1 and HTTP/2?**
HTTP/1.1 processes one request at a time per connection — multiple requests must queue up. HTTP/2 introduces multiplexing, allowing multiple requests and responses to be in-flight simultaneously over a single TCP connection. It also compresses headers and supports server push, significantly improving performance for modern APIs.

**Q: What happens during a TLS handshake?**
The client and server negotiate a TLS version and cipher suite, the server presents its certificate which the client verifies against trusted CAs, then both sides perform a key exchange to derive a shared symmetric encryption key. All subsequent HTTP data is encrypted with that key. TLS 1.3 does this in one round-trip.

**Q: Why should you use 204 instead of 200 for DELETE?**
204 No Content explicitly signals success with no response body, which is semantically correct for a DELETE — the resource is gone, there is nothing to return. Using 200 would imply a body is present. Proper status codes make your API self-documenting and easier to consume correctly.

---

