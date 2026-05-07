# Topic 3: IP Addresses & Ports

## Theory

Every device on a network needs a unique address so data knows where to go. That address is the **IP address**. But an IP address alone only gets you to the right machine — it doesn't tell you which application on that machine should handle the data. That's where **ports** come in.

Think of it this way:
- **IP address** = the building address (gets you to the right machine)
- **Port** = the apartment number (gets you to the right application on that machine)

A single server running ShopSphere might have one IP but dozens of processes listening on different ports — your Spring Boot app on `8080`, PostgreSQL on `5432`, Redis on `6379`, and so on.

---

## Internals — IP Addresses

There are two versions of IP in use today:

**IPv4**
- 32-bit address, written as four octets — `192.168.1.10`
- Supports ~4.3 billion unique addresses
- We ran out of them — hence NAT and IPv6

**IPv6**
- 128-bit address, written in hex — `2001:0db8:85a3::8a2e:0370:7334`
- Supports 340 undecillion addresses — essentially unlimited
- Adoption is growing but IPv4 still dominates most production systems

**Special IP ranges you must know:**
- `127.0.0.1` — loopback address, always means "this machine itself", also called `localhost`
- `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — private IP ranges, not routable on the public internet (used inside VPCs, LANs, Docker networks)
- `0.0.0.0` — means "all available network interfaces on this machine", used when a server wants to accept connections on any interface

**NAT (Network Address Translation)**
- Since IPv4 addresses are scarce, your home router uses NAT — your entire household shares one public IP, and your router maps private IPs (`192.168.x.x`) to that single public IP and back
- Same concept applies in cloud — your EC2 instances have private IPs inside a VPC, and traffic is NATted to public IPs when going out to the internet

---

## Internals — Ports

- A port is a 16-bit number — valid range is **0 to 65535**
- The OS manages ports — when a process wants to receive traffic, it **binds** to a port
- Only one process can bind to a given port at a time on the same machine

**Three port ranges:**

| Range | Name | Description |
|---|---|---|
| 0 – 1023 | Well-Known Ports | Reserved for standard protocols. Require root/admin to bind. |
| 1024 – 49151 | Registered Ports | Used by applications by convention. |
| 49152 – 65535 | Ephemeral Ports | Assigned temporarily by the OS to clients for outbound connections. |

**Ports you must have memorised:**

| Port | Protocol |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 9092 | Kafka |
| 5672 | RabbitMQ |
| 27017 | MongoDB |
| 9200 | Elasticsearch |
| 8080 | Common Spring Boot default |

---

## How IP + Port Work Together

A network connection is uniquely identified by a **5-tuple**:
```
Source IP | Source Port | Destination IP | Destination Port | Protocol (TCP/UDP)
```

Example — your browser opening ShopSphere:
```
Source:      192.168.1.5 : 54231   (your machine, ephemeral port assigned by OS)
Destination: 54.12.45.100 : 443    (ShopSphere server, HTTPS port)
Protocol:    TCP
```

This 5-tuple is what allows a server to handle thousands of simultaneous connections — every pair of (client IP + client ephemeral port) is unique, so the server can track each connection separately even though they all arrive on port 443.

---

## Real-World Example — ShopSphere on Docker Compose

In your ShopSphere Docker Compose setup, every service has its own container with its own internal IP assigned by Docker's virtual network. Here's how ports work:

```yaml
services:
  api-gateway:
    ports:
      - "8080:8080"   # host:container — exposes container port 8080 on your machine's port 8080

  postgres:
    ports:
      - "5432:5432"   # accessible from host for local debugging

  redis:
    ports:
      - "6379:6379"

  elasticsearch:
    ports:
      - "9200:9200"
```

- Inside the Docker network, services talk to each other by **container name + internal port** — e.g. `http://order-service:8081`
- Outside the Docker network (your machine or the internet), they are reached via **localhost + mapped host port**
- In production on Kubernetes, ports are managed by Services and are not directly exposed unless you create a NodePort or LoadBalancer Service

---

## Interview Q&A

**Q: What is the difference between an IP address and a port?**
An IP address identifies a machine on the network. A port identifies a specific process or service running on that machine. You need both to establish a connection — IP gets you to the right server, port gets you to the right application.

**Q: What is a private IP address and why does it exist?**
Private IPs (like `192.168.x.x` or `10.x.x.x`) are not routable on the public internet. They exist because IPv4 addresses are scarce. Devices inside a LAN or VPC use private IPs and access the internet through NAT, which maps them to a single public IP.

**Q: What is `0.0.0.0` and why do you see it in server configs?**
It means "bind to all available network interfaces". When a Spring Boot app listens on `0.0.0.0:8080`, it accepts connections arriving on any network interface of that machine — localhost, internal network, or public interface. It's the standard way to make a server accessible from outside.

**Q: What happens if two processes try to bind to the same port?**
The second one fails with a "port already in use" error. The OS only allows one process to own a given port at a time. This is why you get that error when you try to start your Spring Boot app twice.

**Q: How can a server handle thousands of concurrent connections if it only has one port?**
Because a connection is identified by the full 5-tuple — source IP, source port, destination IP, destination port, and protocol. Every client has a unique source port (ephemeral), so thousands of clients connecting to port 443 are all distinct connections from the OS's perspective.

---

