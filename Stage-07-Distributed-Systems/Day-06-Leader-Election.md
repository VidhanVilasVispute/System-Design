# Stage 7 — Distributed Systems
## Topic 6: Leader Election

---

## Why You Need a Leader

Distributed systems are easier to reason about when **one node is in charge** of a specific responsibility at any given time.

```
Without a leader — every node acts independently:
  Node A: "I'll process this job"
  Node B: "I'll process this job"
  Node C: "I'll process this job"
  → Job processed 3 times. Chaos. 💥

With a leader — one node coordinates:
  Node A (leader): "I'll process this job"
  Node B (follower): "OK, I'll wait"
  Node C (follower): "OK, I'll wait"
  → Job processed exactly once. ✅
```

**Real uses of leader election:**

```
Use Case                          Who Does It
──────────────────────────────────────────────────────────
Kafka partition leader            Kafka Controller (Raft)
Kubernetes control plane          etcd + kube-controller-manager
PostgreSQL primary election       Patroni + etcd/Consul
Scheduled job (run on one node)   Spring's @Scheduled + ShedLock
Shard coordinator                 ZooKeeper / etcd
Distributed lock owner            Redisson (Redis), etcd
```

The challenge: **elect exactly one leader, without a central authority, tolerating failures.**

If you had a central authority to decide — that's a single point of failure. The leader election algorithm IS the mechanism that avoids needing one.

---

## The Core Requirements

A correct leader election must satisfy:

```
Safety:       At most one leader exists at any time
              (no split-brain — two nodes both think they're leader)

Liveness:     A leader is eventually elected
              (system doesn't get stuck forever)

Fault tolerance: If the leader crashes, a new one is elected
```

**Split-brain is the worst failure mode:**

```
Network partition splits cluster into two halves:

  [A, B, C]  ←── partition ───▶  [D, E]

A thinks it's still leader (it was before partition)
D holds an election and wins the right half

Now TWO leaders exist simultaneously.
Both accept writes.
When partition heals → conflicting state.
```

This is why majority quorum is so critical — you need **n/2 + 1** nodes to elect a leader, so only one side of any partition can ever form a majority.

---

## Algorithm 1: Bully Algorithm

The simplest leader election algorithm. Each node has a unique numeric ID. **The node with the highest ID becomes the leader.**

### How It Works

```
Three rules:
1. If a node notices the leader is dead → starts an election
2. It sends Election message to all nodes with HIGHER IDs
3. If no higher node responds → it wins and announces itself
   If a higher node responds → that node takes over the election
```

### Step-by-Step

```
Nodes: 1, 2, 3, 4, 5  — Node 5 is current leader
Node 5 crashes.

─────────────────────────────────────────────────────────────
Step 1: Node 3 detects leader is dead
        Node 3 sends Election to [4, 5]

Step 2: Node 4 responds "I'm alive, step aside"
        Node 5 — no response (crashed)
        Node 4 sends Election to [5]

Step 3: Node 5 — no response
        Node 4 gets no response from higher nodes
        Node 4 declares itself leader
        Node 4 broadcasts Coordinator(4) to all nodes

All nodes now follow Node 4. ✅
─────────────────────────────────────────────────────────────
```

Visualized:

```
Node 1  Node 2  Node 3  Node 4  Node 5(dead)
                  │
                  ├──Election──────▶│
                  │◀──OK────────────┤
                  │                 │──Election──▶ (no response)
                  │                 │
                  │           wins!─┤
                  │◀──Coordinator───┤
         ◀────────┤◀──Coordinator───┤
◀─────────────────┤◀──Coordinator───┤
```

### Problems with Bully

```
1. O(n²) messages — every election floods the network

2. Highest ID bias — if Node 99 keeps crashing and recovering,
   it keeps winning elections even if it's unstable

3. Not partition-aware — in a network split, the higher-ID
   side always wins even if it's the minority
```

Used in: Older systems, simple peer-to-peer networks. Not used in modern production distributed systems.

---

## Algorithm 2: Ring Election

Nodes are arranged in a **logical ring**. Each node only communicates with its neighbor.

```
         1
       /   \
      5     2
      |     |
      4     3
       \   /
         (ring)
```

### How It Works

```
1. Node that detects failure starts election
2. Sends Election(my_id) clockwise around the ring
3. Each node forwards the message, replacing the ID
   if its own ID is higher
4. When the message travels full circle back to originator
   with the highest ID → that node is the winner
5. Winner broadcasts Coordinator message around the ring
```

### Walkthrough

```
Nodes in ring: 1 → 2 → 3 → 4 → 5 → 1 (circular)
Node 3 detects leader is dead, starts election:

  3 sends Election(3) → Node 4
  Node 4: my ID 4 > 3 → forwards Election(4) → Node 5
  Node 5: my ID 5 > 4 → forwards Election(5) → Node 1
  Node 1: my ID 1 < 5 → forwards Election(5) → Node 2
  Node 2: my ID 2 < 5 → forwards Election(5) → Node 3

  Node 3 receives Election(5) back — full circle done
  Node 3 sends Coordinator(5) around the ring
  All nodes learn: Node 5 is the new leader ✅
```

### Problems with Ring

```
1. O(n) messages but O(n) time — slow for large rings
2. If two nodes start elections simultaneously → complex merging
3. If a node fails DURING the election → restart the whole thing
```

Used in: Token Ring networks (legacy), some educational implementations. Still too fragile for production.

---

## Algorithm 3: Raft Leader Election (Production Standard)

You already know this from Topic 5 — but let's look at it purely as a leader election mechanism.

```
Key insight: randomized election timeouts eliminate the need
for explicit "who has the highest ID" logic.

Each follower waits a RANDOM timeout (150–300ms).
The first one to timeout becomes candidate.
Because timeouts are random → usually only one fires first.
That node requests votes, gets majority, becomes leader.
```

### Why Randomness Works

```
5 nodes, random timeouts:
  Node A: timeout = 152ms  ← fires first
  Node B: timeout = 203ms
  Node C: timeout = 178ms
  Node D: timeout = 267ms
  Node E: timeout = 231ms

Node A fires, sends RequestVote to B, C, D, E
B, C, D, E: haven't timed out yet → grant vote
Node A wins with 5/5 votes before anyone else even tries

Probability of tie: very low (random range is wide)
If tie: both candidates timeout again with new random values → resolves
```

### What Makes Raft's Election Safe (No Split-Brain)

```
Rule: A node only votes for a candidate whose log is
      at least as up-to-date as its own.

This prevents a stale node (missed entries) from becoming
leader and overwriting committed data.

Combined with majority requirement:
→ Any two majorities share at least one node
→ That shared node only votes once
→ Only one leader can win any given term
```

---

## Algorithm 4: etcd-Based Leader Election (Practical Pattern)

In microservices, you often don't implement leader election yourself. You use **etcd or ZooKeeper** as the coordination backbone.

### How It Works

```
Multiple instances of your service compete to write a key in etcd:
  Key:   /shopsphere/locks/scheduler-leader
  Value: {nodeId: "pod-abc123", hostname: "..."}
  TTL:   10 seconds  ← key expires if not refreshed

First instance to successfully write → IS THE LEADER
Other instances: watch the key, wait for it to disappear

Leader continuously refreshes the TTL (heartbeat)
If leader crashes → TTL expires → key disappears
Other instances detect this → race to write → new leader emerges
```

Visualized:

```
Pod A ──write(/leader, pod-a, TTL=10s)──▶ etcd  ✅ (wins)
Pod B ──write(/leader, pod-b, TTL=10s)──▶ etcd  ❌ (key exists)
Pod C ──write(/leader, pod-c, TTL=10s)──▶ etcd  ❌ (key exists)

Pod A  ──refresh every 5s──▶ etcd  ✅ (keeps leadership)

Pod A crashes → stops refreshing → TTL expires → key deleted

Pod B watches key → sees deletion → write(/leader, pod-b, TTL=10s) ✅
Pod C watches key → sees deletion → write(/leader, pod-c, TTL=10s) ❌

New leader: Pod B ✅
```

**This is exactly how Kubernetes leader election works** — the kube-controller-manager, kube-scheduler all use this pattern via the `coordination.k8s.io/v1` Lease API (which is backed by etcd).

---

## ShedLock — Leader Election for Spring Boot Jobs

In ShopSphere, some services run scheduled tasks — cleanup jobs, report generation, cache warming. In a multi-replica deployment, you don't want all 3 replicas running the same job simultaneously.

**ShedLock** is the Spring Boot-native solution. It uses your existing database as the coordination store.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.0</version>
</dependency>
```

```sql
-- Flyway migration: V3__shedlock.sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL,
    PRIMARY KEY (name)
);
```

```java
@Configuration
public class ShedLockConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()   // use DB clock, not app server clock
                .build()
        );
    }
}
```

```java
// order-service: cleanup expired orders — only one pod should run this
@Component
@EnableScheduling
public class OrderCleanupJob {

    @Scheduled(cron = "0 0 2 * * *")   // 2 AM daily
    @SchedulerLock(
        name = "OrderCleanupJob",
        lockAtLeastFor = "PT5M",    // hold lock minimum 5 minutes
        lockAtMostFor  = "PT30M"    // release after 30 min even if pod hangs
    )
    public void cleanupExpiredOrders() {
        // Only ONE pod across the cluster executes this
        // Others try to acquire lock, fail, and skip silently
        orderRepository.deleteExpiredOrders();
    }
}
```

```
How ShedLock works internally:
────────────────────────────────────────────────────────────
Pod A: INSERT INTO shedlock (name, lock_until, locked_at, locked_by)
       VALUES ('OrderCleanupJob', now()+30min, now(), 'pod-a')
       → succeeds ✅ → runs the job

Pod B: INSERT INTO shedlock ... (same name)
       → CONFLICT (primary key violation) ❌ → skips job

Pod A finishes → UPDATE lock_until = now() (releases lock early)
────────────────────────────────────────────────────────────
```

---

## The Split-Brain Problem in Detail

This deserves its own deep look because it's the most dangerous failure mode.

```
Scenario: 5-node Kafka cluster, Node 1 is leader for partition 0
Network partition occurs:

  Partition A: [Node1, Node2]       Partition B: [Node3, Node4, Node5]

Node1 thinks it's still leader (it was before partition)
  → Accepts writes from producers → appended to its log

Partition B has majority (3/5)
  → Elects Node3 as new leader for term 2
  → Node3 accepts writes from producers → appended to its log

BOTH nodes are accepting writes for the same partition! 💥

Partition heals:
  Node1's log and Node3's log have DIVERGED
  Which writes are valid? Which are lost?
```

### How Raft Prevents Split-Brain

```
When partition heals:
  Node1 sees term=2 messages from Node3
  Node1 has term=1 → it's stale
  Node1 STEPS DOWN immediately, becomes follower
  Node1 rolls back uncommitted entries that conflict with Node3's log
  Node1 syncs from Node3

Writes that Node1 accepted but weren't committed (not majority-replicated)
→ LOST (client got an error or timeout, not a confirmation)

Writes that were committed (majority replicated before partition)
→ ALWAYS SAFE (by Raft's log completeness property)
```

This is why Raft's majority requirement is so fundamental — **writes are only confirmed after majority replication**, so a minority partition leader can never have committed entries the majority doesn't have.

---

## ShopSphere — Where Leader Election Matters

```
Service / Component           Leader Election Mechanism
──────────────────────────────────────────────────────────────────
Kubernetes scheduler          Lease API (etcd) — one scheduler pod active
Kubernetes controller-manager Lease API (etcd) — one controller active
Kafka partition leaders        KRaft Raft election — one broker per partition
PostgreSQL primary            Patroni + etcd — one primary per service DB
Scheduled jobs (cleanup etc.) ShedLock + PostgreSQL — one pod runs job
Redis (Sentinel mode)         Sentinel quorum — one Redis primary
Elasticsearch primary shards  Zen discovery / Raft — one primary shard per index
```

In a multi-replica `order-service` deployment on Kubernetes:

```
3 replicas of order-service:
  pod-1, pod-2, pod-3

Each pod runs @Scheduled jobs
Without ShedLock:
  All 3 fire cleanup at 2AM → 3x duplicate work, potential conflicts

With ShedLock:
  pod-1 acquires lock → runs cleanup
  pod-2 tries lock   → conflict → skips
  pod-3 tries lock   → conflict → skips
```

---

## Lease-Based vs Quorum-Based Election

```
Lease-Based (etcd/ShedLock):
  Simple: write a key with TTL, first writer wins
  Fast: single write operation
  Risk: if leader is slow (GC pause, high load), TTL might expire
        while leader still thinks it holds the lease
        → brief split-brain possible during GC pauses
  Used for: scheduled jobs, low-stakes coordination

Quorum-Based (Raft):
  Complex: full round-trip voting protocol
  Slower: requires majority acknowledgment
  Safe: mathematically impossible to have two leaders in same term
  Used for: database primaries, consensus-critical coordination
```

---

## Interview Angles 🎯

**Q: What is split-brain and why is it dangerous?**
> Split-brain is when two nodes simultaneously believe they are the leader. It happens during network partitions when both sides elect a leader independently. It's dangerous because both leaders accept writes, leading to diverged state. When the partition heals, one side's writes must be discarded. Raft prevents it by requiring majority quorum — only one side of any partition can form a majority.

**Q: How does Kubernetes ensure only one scheduler runs at a time?**
> Using the Lease API backed by etcd. The active scheduler pod holds a Lease object with a TTL and renewals it every few seconds. If the pod crashes, the lease expires, and other scheduler pods race to acquire it. The first one to successfully write the lease becomes active. This is leader election via distributed lock with TTL.

**Q: Why does Raft use randomized timeouts for election?**
> To avoid all followers timing out simultaneously and starting competing elections. With random timeouts in a range like 150–300ms, typically only one node times out first and wins the election before others even start. If two do tie, they time out again with new random values, resolving the conflict.

**Q: How would you prevent a Spring Boot scheduled job from running on all 3 pods simultaneously?**
> Use ShedLock. It inserts a row into a `shedlock` table before the job runs. Since the row has the job name as the primary key, only one pod's insert succeeds — others get a constraint violation and skip. The row has a `lock_until` timestamp so the lock is automatically released even if the pod crashes.

**Q: What's the difference between leader election and distributed locking?**
> Conceptually similar but different scope. A distributed lock is acquired for a short operation (process this request, run this transaction) and released immediately after. Leader election is a longer-term role assignment — the leader holds its role until it crashes or voluntarily steps down, and election involves detecting failure and re-electing. Leader election IS essentially a long-duration distributed lock with automatic re-acquisition on failure.

---

Say **next** for **Topic 7: Quorum Reads/Writes** — how Cassandra and DynamoDB tune consistency vs availability with a single number 🚀
