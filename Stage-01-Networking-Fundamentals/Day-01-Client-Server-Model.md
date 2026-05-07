

# Topic 1: Client-Server Model

## Theory

The client-server model is the foundational architecture of the internet. Every time you open an app, watch a video, or send a message — you are a **client** making a request, and some machine somewhere is a **server** responding to it.

- **Client** — the one who initiates the request. It is always the one asking. Your browser, your mobile app, Postman — all clients.
- **Server** — the one who listens and responds. It never starts the conversation. It just waits, processes, and replies.
- This is a **request-response** model — the client sends a request, the server sends back a response.
- The server is always **on** — it is a long-running process bound to a port, waiting for incoming connections.

Why does this model exist? Because we need a clear separation between who owns the data and logic (server) and who consumes it (client). Without this separation, every device would need all the data locally, which is impractical.

---

## Internals

Here's what actually happens when your app makes a request:

- Your client opens a **socket** — a combination of IP address + port that represents a communication endpoint.
- It connects to the server's IP and a specific port (e.g., port 443 for HTTPS).
- A **TCP connection** is established via a 3-way handshake (SYN → SYN-ACK → ACK).
- The client sends an HTTP request over this connection.
- The server processes it and writes an HTTP response back over the same connection.
- Connection is either kept alive (HTTP keep-alive) or closed.

Key points:
- The server binds to a **well-known port** (80 for HTTP, 443 for HTTPS, 5432 for PostgreSQL).
- The client uses a **random ephemeral port** (e.g., 52341) assigned by the OS for that session.
- A server can handle thousands of clients simultaneously using threads, processes, or non-blocking I/O (event loop).

---

## Real-World Example — ShopSphere

When a user opens ShopSphere and searches for a product:

- The **React frontend** (client) sends `GET /api/search?q=shoes` to your API Gateway.
- The **API Gateway** (now acting as a client itself) forwards the request to your **Search Service**.
- The Search Service queries **Elasticsearch**, gets results, and sends the response back up the chain.
- The frontend receives the response and renders the product list.

Notice that in distributed systems, a service is both a server (to whoever calls it) and a client (to whatever it calls). The client-server model is recursive throughout your entire architecture.

---

## Interview Q&A

**Q: What is the difference between a client and a server?**
A client initiates the request, a server listens and responds. The key is who starts the conversation — it is always the client.

**Q: Can a server also be a client?**
Yes, always in microservices. Your Order Service is a server to the API Gateway, but it is a client when it calls the Payment Service or publishes to Kafka.

**Q: What happens if the server goes down while the client is waiting?**
The TCP connection will time out. The client receives a connection refused or timeout error. This is why we have retry logic, circuit breakers, and health checks — all of which we will cover in later stages.

**Q: What is the difference between client-server and peer-to-peer?**
In client-server, roles are fixed — one side always serves. In peer-to-peer (BitTorrent, WebRTC), every node can be both client and server simultaneously. P2P is more resilient but harder to control and secure.

---

