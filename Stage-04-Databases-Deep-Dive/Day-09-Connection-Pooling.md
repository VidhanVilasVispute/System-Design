# Stage 4 — Databases Deep Dive
## Topic 9: Connection Pooling — Internals, HikariCP, PgBouncer, Sizing, Failure Modes

---

## 1. What a Database Connection Actually Is

Before pooling makes sense, understand what you're pooling.

### At the OS Level

When your Spring Boot app calls `DriverManager.getConnection(url, user, pass)`:

```
App Process
    │
    │  1. TCP SYN → PostgreSQL host:5432
    │  2. TCP SYN-ACK ← 
    │  3. TCP ACK →        (TCP 3-way handshake)
    │
    │  4. SSL ClientHello →
    │  5. SSL ServerHello, Certificate ←
    │  6. SSL ClientKeyExchange →       (TLS handshake if SSL enabled)
    │
    │  7. StartupMessage (user, database) →
    │  8. AuthenticationMD5Password ←
    │  9. PasswordMessage (hashed) →
    │  10. AuthenticationOK ←
    │  11. ParameterStatus (timezone, encoding...) ←
    │  12. BackendKeyData (PID, secret key) ←
    │  13. ReadyForQuery ←              (connection established)
```

**13 network round trips** before you can send your first SQL. At 1ms per round trip on a local network — 13ms just to establish a connection. At 5ms (cross-AZ) — 65ms per connection.

### At the PostgreSQL Level

PostgreSQL uses a **process-per-connection model** (not threads — full OS processes):

```bash
# What you see on the DB server:
ps aux | grep postgres

postgres  1234  postgres: checkpointer
postgres  1235  postgres: background writer
postgres  1236  postgres: walwriter
postgres  1237  postgres: autovacuum launcher
postgres  1238  postgres: backend user=app db=shopsphere   ← one per connection
postgres  1239  postgres: backend user=app db=shopsphere
postgres  1240  postgres: backend user=app db=shopsphere
...
```

Each backend process:
- Consumes **5–10MB RAM** for its private memory (work_mem, stack, etc.)
- Holds shared memory locks while active
- Has its own CPU scheduling overhead

At 500 connections: 500 processes × 8MB = **4GB RAM** just for connection overhead, before any actual query processing.

PostgreSQL's `max_connections` defaults to 100. Exceeding it throws:
```
FATAL: sorry, too many clients already
```

---

## 2. HikariCP Internals — How the Pool Works

HikariCP is the fastest JDBC connection pool. Spring Boot auto-configures it.

### Pool Lifecycle

```
Application Startup:
  HikariCP creates minimumIdle connections immediately
  Each connection: full TCP+SSL+auth handshake (done once, amortized)
  Connections placed in the pool (idle state)

Request comes in:
  Thread calls pool.getConnection()
  Pool finds an idle connection → marks it "in-use" → returns it
  Thread executes SQL
  Thread calls connection.close() → returns to pool (NOT closed!)
  Pool marks connection "idle" again

Pool at maximumPoolSize:
  All connections in use
  New request waits in queue for connectionTimeout ms
  If timeout exceeded → HikariCP throws SQLTimeoutException
```

### HikariCP's ConcurrentBag

HikariCP uses a lock-free data structure called `ConcurrentBag` internally — not a simple queue. It has three levels:

1. **Thread-local list** — connections last used by this specific thread are checked first. Cache locality — thread likely reuses the same connection, which has warm caches on the DB side.
2. **Shared list** — connections available to any thread.
3. **Waiting queue** — if no connection available, thread parks here until one is returned.

This is why HikariCP is faster than older pools (DBCP, C3P0) — the thread-local optimization reduces contention on the shared structure.

### Key Configuration Deep Dive

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000      # ms to wait for a connection from pool
      idle-timeout: 600000           # ms before idle connection is closed (10min)
      max-lifetime: 1800000          # ms before connection is forcibly recycled (30min)
      keepalive-time: 60000          # ms between keepalive pings (prevents firewall drops)
      connection-test-query: SELECT 1 # used if JDBC4 isValid() not supported
      pool-name: ShopSphere-Order-Pool
      leak-detection-threshold: 5000 # warn if connection held > 5s (detect leaks)
```

**`max-lifetime` is critical:** PostgreSQL's TCP stack, firewalls, and NAT gateways silently kill idle connections after some timeout (often 30–60 minutes). If your app holds a connection past that point and tries to use it, you get a `broken pipe` error. `max-lifetime` (set lower than the network timeout) forces HikariCP to proactively recycle connections before they go stale.

**`leak-detection-threshold`:** If a thread holds a connection longer than this value, HikariCP logs a stack trace showing where the connection was acquired. Invaluable for finding code that forgot to return connections (missing `finally` blocks, unclosed `ResultSet`s).

### Connection Validation

Before returning a connection to a thread, HikariCP optionally validates it:

```
isValid(1) → sends a lightweight ping to DB
           → if fails, discard this connection, get another
           → if passes, return to thread
```

This adds ~1ms per borrow but prevents "dead connection" errors reaching your application code. Worth enabling in production.

---

## 3. The Pool Sizing Problem

Most engineers set `maximum-pool-size` too high. This is counterintuitive.

### Why Bigger Pool ≠ Better Performance

PostgreSQL processes queries using CPU. A 4-core DB server can execute 4 things in parallel. Having 200 connections all firing queries simultaneously doesn't make those queries run faster — it creates:

- **Process scheduling overhead:** OS spends time context-switching between 200 backend processes
- **Lock contention:** More concurrent transactions → more lock waits
- **Memory pressure:** 200 processes × 8MB = 1.6GB just for backends
- **Cache thrashing:** Each process evicts others' pages from `shared_buffers`

HikariCP's own benchmarks show: **a pool of 10 connections outperforms a pool of 100** on the same hardware beyond a certain concurrency level.

### The Formula

```
pool_size = (num_cores * 2) + effective_spindle_count
```

For a 4-core DB server with SSD (spindle count ≈ 1):
```
pool_size = (4 × 2) + 1 = 9  ← per app instance
```

For a 16-core DB server with NVMe:
```
pool_size = (16 × 2) + 1 = 33
```

This formula is per application instance. With 10 Order Service instances, each with pool=9, that's 90 connections to PostgreSQL — still manageable.

### The Real Calculation

```
Total DB connections = num_app_instances × pool_size_per_instance

Constraint: Total DB connections < PostgreSQL max_connections - 10
            (leave headroom for superuser connections, replication, maintenance)

Example: max_connections=200, 5 services × 3 instances × pool=10 = 150 connections
         150 < 190 ✅
```

---

## 4. PgBouncer — Connection Pooling at Infrastructure Level

HikariCP pools connections inside the application process. PgBouncer pools at the network level — sits between all app instances and PostgreSQL.

```
Without PgBouncer:
  10 services × 5 instances × pool=20 = 1000 PostgreSQL connections

With PgBouncer:
  10 services × 5 instances × pool=20 = 1000 connections to PgBouncer
  PgBouncer → PostgreSQL: 30 connections (you control this)
  PostgreSQL sees only 30 backends
```

### PgBouncer Modes — The Critical Difference

**Session Mode:**
A server connection is assigned to a client for the entire session duration. Equivalent to no pooling from PostgreSQL's perspective — just adds a proxy hop. Only benefit: hides TCP connection setup from client.

**Transaction Mode:**
A server connection is held only for the duration of one transaction. Returned to pool between transactions.

```
Client sends: BEGIN
  → PgBouncer acquires server connection from pool
Client sends: INSERT ...
Client sends: COMMIT
  → PgBouncer releases server connection back to pool

Client is now idle (between transactions)
  → Server connection available for other clients
```

This is the powerful mode. 1000 app-side connections sharing 30 server connections — possible because most clients are idle between transactions most of the time.

**Limitations of Transaction Mode:**
These PostgreSQL features require session-level state — they break in transaction mode:

| Feature | Why it breaks |
|---|---|
| `SET LOCAL` | Session variable lost when connection returned |
| `LISTEN/NOTIFY` | Session-level subscription |
| Named prepared statements | Session-level (use protocol-level unnamed statements) |
| Advisory locks | Session-scoped locks |
| `WITH HOLD` cursors | Session-level cursor |

Spring Boot + JPA with PgBouncer in transaction mode: works correctly for standard queries and transactions. Avoid session-level features or configure PgBouncer to route those connections differently.

**Statement Mode:**
Connection returned after every single SQL statement. Completely breaks multi-statement transactions. Almost never used.

### PgBouncer Configuration

```ini
[databases]
shopsphere = host=postgres-primary port=5432 dbname=shopsphere

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000        # max app-side connections to PgBouncer
default_pool_size = 30        # server connections to PostgreSQL per db/user pair
reserve_pool_size = 5         # extra connections for emergencies
reserve_pool_timeout = 3      # seconds to wait before using reserve pool
server_idle_timeout = 600     # close idle server connections after 10min
server_lifetime = 3600        # recycle server connections every 1hr
```

---

## 5. Failure Modes

### Pool Exhaustion

All connections in use. New requests queue up. Queue fills. `connectionTimeout` expires.

```
Spring Boot throws:
  Unable to acquire JDBC Connection; 
  nested exception is java.sql.SQLTimeoutException: 
  Timeout of 30000ms exceeded waiting for connection
```

**Cause:** Slow queries holding connections too long, connection leak (thread holding connection and not returning it), traffic spike exceeding pool capacity.

**Detection:** Monitor `HikariCP.pool.Pending` metric — number of threads waiting for a connection. Alert if > 0 for sustained period. Also monitor `HikariCP.pool.ActiveConnections` approaching `maximum-pool-size`.

**Fix:** 
- Short-term: increase pool size (but understand the DB side impact)
- Medium-term: find and fix slow queries, fix leaks
- Long-term: add read replicas, add caching, reduce DB load

### Connection Leak

A thread borrows a connection and never returns it — an exception was thrown, the `finally` block was missing, or a `ResultSet` wasn't closed.

```java
// BUG: if exception thrown between getConnection() and close(), leak occurs
Connection conn = pool.getConnection();
ResultSet rs = conn.createStatement().executeQuery("SELECT ...");
// exception here → conn never returned
conn.close();
```

With Spring `@Transactional`, this is managed automatically — Spring opens and closes connections via the transaction manager. Leaks happen in manual JDBC code.

**Detection:** `leak-detection-threshold` in HikariCP logs the acquisition stack trace if a connection is held past the threshold. Watch for logs like:
```
Connection leak detection triggered for ... stack trace:
  at com.shopsphere.OrderRepo.findAll(OrderRepo.java:42)
```

### Stale Connection — Silent TCP Kill

Firewalls, AWS security groups, and NAT gateways silently drop idle TCP connections after a timeout (commonly 350 seconds on AWS). If HikariCP's `max-lifetime` is longer than this, it tries to use a dead connection and gets a broken pipe exception.

**Fix:** Set `max-lifetime` below the network timeout. On AWS: set `max-lifetime=290000` (just under 5 minutes). Also enable `keepalive-time` to send TCP keepalives on idle connections.

### PostgreSQL `max_connections` Exceeded

If total connections from all app instances + poolers exceed `max_connections`, PostgreSQL rejects new connections:
```
FATAL: sorry, too many clients already
```

**Fix:** Always deploy PgBouncer in front of PostgreSQL so PostgreSQL only sees pooler connections. Set `max_connections` conservatively (100–200). Let PgBouncer handle the fan-out.

---

## 6. Monitoring Connection Pool Health

Key metrics to expose (HikariCP → Micrometer → Prometheus → Grafana):

```java
// Auto-exported by Spring Boot Actuator + HikariCP + Micrometer:
hikaricp_connections_active     // connections currently in use
hikaricp_connections_idle       // connections waiting in pool
hikaricp_connections_pending    // threads waiting for a connection ← most important
hikaricp_connections_timeout_total  // cumulative connection timeouts
hikaricp_connection_acquire_nanos   // time to get a connection from pool
hikaricp_connection_usage_millis    // time connection was held by a thread
hikaricp_connection_creation_millis // time to create a new connection
```

**Alert thresholds:**
- `connections_pending > 0` for > 30 seconds → pool exhaustion imminent
- `connections_timeout_total` increasing → pool is exhausted, requests failing
- `connection_acquire_nanos` p99 > 100ms → pool under pressure
- `connections_active == maximum-pool-size` for sustained period → increase pool or reduce DB load

---

## 7. ShopSphere Configuration

```yaml
# Order Service — write-heavy, needs fast connection acquisition
spring.datasource.hikari:
  pool-name: OrderService-Primary
  maximum-pool-size: 10
  minimum-idle: 3
  connection-timeout: 5000       # fail fast — don't queue for 30s
  max-lifetime: 290000           # below AWS NAT 350s timeout
  keepalive-time: 60000
  leak-detection-threshold: 3000 # warn if connection held > 3s

# Product Service — read-heavy, mostly replica reads
spring.datasource.replica.hikari:
  pool-name: ProductService-Replica
  maximum-pool-size: 15          # slightly larger, read queries
  minimum-idle: 5
  connection-timeout: 10000
  max-lifetime: 290000
```

**PgBouncer in front:**
```
Order Service (3 instances × pool=10)   = 30 app→pgbouncer connections
Product Service (5 instances × pool=15) = 75 app→pgbouncer connections
... other services ...
Total app→pgbouncer: ~300 connections

PgBouncer→PostgreSQL:
  primary pool: 25 server connections
  replica pool: 40 server connections

PostgreSQL max_connections = 100 (comfortable headroom)
```

---

## 8. Interview Q&A

**Q: Why does PostgreSQL use one process per connection instead of threads?**
A: Historical design decision rooted in stability and isolation — process boundaries prevent one connection's memory corruption from affecting others. Threads share memory space; a bug in one thread can corrupt another's query execution. The trade-off is the high per-connection overhead (5–10MB, OS process scheduling). This is precisely why connection pooling is non-optional at scale — the process model doesn't tolerate thousands of concurrent connections.

**Q: What's the difference between HikariCP and PgBouncer?**
A: HikariCP pools connections inside the application process — each app instance maintains its own pool. PgBouncer pools at the network level, sitting between all app instances and PostgreSQL. HikariCP reduces connection setup overhead per instance. PgBouncer limits total connections to PostgreSQL regardless of how many app instances exist. In production you use both: HikariCP for low-latency connection borrowing within an instance, PgBouncer to cap total DB connections as the fleet scales.

**Q: Why is transaction-mode pooling more efficient than session-mode?**
A: In session mode, a server connection is tied to a client for their entire session — while the client is idle between transactions, the server connection is unavailable to others. In transaction mode, the server connection is returned to the pool immediately after COMMIT or ROLLBACK. A web app spends most time outside transactions (waiting for the client, processing application logic) — transaction mode lets that connection serve other clients during that idle time. This is why 1000 app connections can share 30 server connections in transaction mode.

**Q: What causes connection pool exhaustion and how do you fix it?**
A: Causes: slow queries holding connections too long (a 10-second query holds a connection 10 seconds, blocking 10 other requests); traffic spikes exceeding pool capacity; connection leaks (code that doesn't return connections); cascade failures (a slow downstream service causes all threads to pile up waiting). Fix: instrument `connections_pending` — identify the cause. For slow queries: add indexes, add caching. For leaks: use `leak-detection-threshold` to find the culprit. For traffic spikes: add read replicas, add caching, rate-limit upstream.

**Q: You set `maximum-pool-size=100` to handle more traffic. Performance gets worse. Why?**
A: Pool size too large causes DB-side contention. PostgreSQL is running 100 backend processes competing for CPU, locks, and `shared_buffers` cache pages. Context switching overhead increases. Lock wait chains grow. The optimal pool size is typically `(cores × 2) + spindles` per app instance. Beyond that, adding more connections hurts. The real solutions for higher throughput are: more app instances each with a right-sized pool, read replicas, caching, or query optimization.

---

Ready for **Topic 10: ACID vs BASE — strong consistency vs eventual consistency, the real trade-off behind SQL vs NoSQL**?
