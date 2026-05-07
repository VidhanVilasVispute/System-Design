#  Topic 2: DNS Resolution

## Theory

DNS stands for **Domain Name System**. It is essentially the phonebook of the internet.

Computers communicate using IP addresses like `142.250.182.46`. But humans remember names like `google.com`. DNS is the system that **translates a domain name into an IP address** so your machine knows where to actually send the request.

Without DNS, you would have to memorise IP addresses for every website you visit. DNS abstracts that away entirely.

Key terms to know:
- **Domain Name** — the human-readable address, e.g. `api.shopsphere.com`
- **IP Address** — the machine-readable address, e.g. `54.12.45.100`
- **DNS Record** — an entry in DNS that maps a name to something (IP, another name, mail server, etc.)
- **DNS Resolver** — the component that does the lookup on your behalf
- **TTL (Time To Live)** — how long a DNS response can be cached before it must be re-fetched

---

## Internals — What Actually Happens When You Type a URL

This is a step-by-step breakdown of a full DNS resolution:

**1. Browser Cache** — the browser first checks if it already resolved this domain recently and cached the result. If yes, done.

**2. OS Cache** — if not in browser cache, the OS checks its own DNS cache and the local `/etc/hosts` file.

**3. Recursive Resolver** — if still not found, your OS sends the query to a **Recursive Resolver** — usually provided by your ISP or a public one like Google (`8.8.8.8`) or Cloudflare (`1.1.1.1`). This resolver does the heavy lifting on your behalf.

**4. Root Name Server** — the resolver asks a **Root Name Server** — "who knows about `.com` domains?" There are only 13 sets of root servers in the world. The root server responds with the address of the **TLD Name Server**.

**5. TLD Name Server** — the resolver asks the **Top-Level Domain server** (the `.com` TLD server) — "who knows about `shopsphere.com`?" It responds with the **Authoritative Name Server** for that domain.

**6. Authoritative Name Server** — this is the server that actually holds the DNS records for `shopsphere.com`. The resolver asks it for the IP of `api.shopsphere.com` and gets back an A record with the IP address.

**7. Response Cached and Returned** — the resolver caches the result per the TTL, sends it back to your OS, which sends it back to your browser. Now the browser can open a TCP connection to that IP.

```
Browser → Recursive Resolver → Root NS → TLD NS → Authoritative NS
                                                          ↓
Browser ←————————————— IP Address returned ——————————————
```

---

## Common DNS Record Types

- **A Record** — maps a domain to an IPv4 address. Most common.
- **AAAA Record** — maps a domain to an IPv6 address.
- **CNAME Record** — maps a domain to another domain name (alias). e.g. `www.shopsphere.com → shopsphere.com`
- **MX Record** — specifies the mail server for a domain.
- **TXT Record** — arbitrary text, used for domain verification and SPF/DKIM email security.
- **NS Record** — specifies which name servers are authoritative for this domain.

---

## Real-World Example — ShopSphere

Say your ShopSphere backend is deployed on AWS. You have:

- `api.shopsphere.com` → points via **A record** to your AWS Load Balancer IP
- `cdn.shopsphere.com` → points via **CNAME** to your CloudFront distribution URL
- `mail.shopsphere.com` → points via **MX record** to your email provider (SendGrid)

When a user hits `api.shopsphere.com` from their phone:
- DNS resolves it to your Load Balancer's IP
- The Load Balancer routes the request to one of your API Gateway instances
- Your microservices handle the rest

Also important — when you do a **rolling deployment** and switch your backend to a new server IP, you update the A record. The TTL determines how long old clients keep hitting the old IP before they pick up the new one. Low TTL = faster propagation, more DNS queries. High TTL = slower propagation, less load on DNS.

---

## Interview Q&A

**Q: What is DNS and why do we need it?**
DNS translates human-readable domain names into machine-readable IP addresses. Without it, every client would need to know the raw IP of every server, which is unmanageable at scale.

**Q: What is the difference between a Recursive Resolver and an Authoritative Name Server?**
A Recursive Resolver does the work of finding the answer — it queries multiple servers on your behalf. An Authoritative Name Server is the final source of truth — it holds the actual DNS records for a domain.

**Q: What is TTL in DNS and why does it matter in system design?**
TTL is how long a DNS response is cached. In system design it matters during failovers and deployments — if your TTL is 3600 seconds and you switch your server IP, some clients will keep hitting the old IP for up to an hour. A low TTL like 60 seconds gives you faster failover at the cost of more DNS query traffic.

**Q: What is a CNAME and when would you use it?**
A CNAME maps a domain to another domain name instead of directly to an IP. You use it when the target IP can change (like a CDN or load balancer URL) and you don't want to update every A record manually.

**Q: Can DNS be a single point of failure?**
Yes, if you use a single DNS provider with no redundancy. Production systems use multiple authoritative name servers (AWS Route 53, Cloudflare) which are globally distributed and highly available by design.

---
