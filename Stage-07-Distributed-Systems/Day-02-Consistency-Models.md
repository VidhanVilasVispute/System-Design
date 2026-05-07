# Stage 7 — Distributed Systems
## Topic 2: Consistency Models

---

## What Is a Consistency Model?

When you write data to a distributed system and then read it back — **what do you get?**

A **consistency model** is a contract between the database and the developer:

> *"If you write X, here's what I guarantee you'll read — and when."*

Different systems make different guarantees. Understanding them lets you pick the right tool and reason about bugs that aren't bugs — they're just the system behaving as promised.

---

## The Spectrum

```
STRONGEST                                                    WEAKEST
    │                                                            │
    ▼                                                            ▼
Linearizability → Sequential → Causal → Read-Your-Writes → Eventual
(Strong)                                                  Consistency
```

More consistency = **slower, harder to scale**
Less consistency = **faster, more available, but surprising behavior**

---

## 1. 🔴 Strong Consistency (Linearizability)

**The contract:**
> Every read reflects the most recent write. Period. The system behaves as if there's only one copy of the data.

```
Timeline:
──────────────────────────────────────────────────
Thread A:  write(x = 10) ──────────────────▶ ✅
Thread B:             read(x) ──▶ gets 10 (never gets old value)
──────────────────────────────────────────────────
```

Even if 3 replicas exist internally, the system **hides that from you**. It looks like one machine.

**How it works under the hood:**
- Writes go to a **leader**, which replicates to followers
- Read is only confirmed after **majority acknowledges** the write
- This requires network round trips → **latency cost**

**Real systems:**
```
etcd         → Linearizable (used by Kubernetes for all state)
ZooKeeper    → Linearizable writes
PostgreSQL   → Linearizable within a single node
Google Spanner → Linearizable globally (uses TrueTime atomic clocks)
```

**ShopSphere example:**
```
inventory-service: stock = 1 (last item)

User1 → writes stock = 0         ✅
User2 → reads stock immediately  → gets 0 (never oversells)

This is what you need for flash sales, seat booking, payment deduction.
```

**Cost:** You must wait for replicas to sync before confirming a write.
During a partition → system goes **unavailable** rather than serve stale data (CP).

---

## 2. 🟡 Sequential Consistency

**The contract:**
> All operations appear in **some** sequential order. Every node agrees on that order. But it might not match real wall-clock time.

```
Reality (wall clock):
  Node A: write(x=1) at 10:00:00
  Node B: write(x=2) at 10:00:01

Sequential consistency allows:
  All nodes see: x=2 then x=1 (as long as ALL see the same order)

Linearizability would NOT allow this — it must respect real time.
```

Weaker than linearizability, but still gives a **global agreed-upon order**.

**Used in:** Some CPU memory models, older distributed databases.
Less common as an explicit guarantee in modern systems — usually you get either linearizable or something weaker.

---

## 3. 🟡 Causal Consistency

**The contract:**
> Operations that are **causally related** must be seen in order.
> Concurrent (unrelated) operations can be seen in any order.

```
Causal relationship example:
──────────────────────────────────────────────────────
User posts a comment: "Great product!"        ← write A
User posts a reply:   "Thanks for the review" ← write B (caused by A)

Causal consistency guarantees:
  Anyone who sees B must also have seen A first.
  You'll never see the reply before the original comment.
──────────────────────────────────────────────────────
```

But two unrelated writes — say, two independent reviews from different users — can appear in any order.

**How it's tracked:** Vector clocks (Topic 4!) — each write carries a version vector that encodes its causal dependencies.

**Real systems:**
```
MongoDB   → "causal consistency" sessions (explicit opt-in)
DynamoDB  → supports causal consistency with version vectors
CosmosDB  → one of its 5 consistency levels
```

**ShopSphere example:**
```
review-service:
  User writes Review (id=42)
  User writes Reply to Review (id=42)

Causal consistency ensures:
  Any reader who sees the reply also sees the original review.
  No "orphan reply" appearing before its parent.
```

---

## 4. 🟡 Read-Your-Writes (Session Consistency)

**The contract:**
> After you write something, **you** will always read your own write.
> Other users might still see stale data.

```
──────────────────────────────────────────────────────
User updates their profile picture.
User immediately visits their profile page.

Read-Your-Writes guarantees:
  The user sees THEIR new picture.
  Other users might still see the old one for a short while.
──────────────────────────────────────────────────────
```

**How it's implemented:**
- Route reads from the same user to the same replica (sticky sessions)
- Or: attach a **write timestamp** to the user's session token; replica only serves the read if it has caught up to at least that timestamp

**Real systems:**
```
DynamoDB  → "session consistency" mode
Cassandra → achieved via token-aware routing
Most web apps → achieved by reading from primary after write (for that user's session)
```

**ShopSphere example:**
```
user-service:
  User changes their email → write goes to primary
  User is redirected to profile page → read must see the new email

Without read-your-writes:
  User would see their OLD email right after changing it → confusing bug
```

This isn't a bug in the code — it's the consistency model behaving as designed. You need to explicitly ensure read-your-writes for user-facing flows.

---

## 5. 🟢 Eventual Consistency

**The contract:**
> If no new writes happen, all replicas will **eventually** converge to the same value.
> Reads may return stale data in the meantime.
> No guarantee on **how long** "eventually" takes.

```
──────────────────────────────────────────────────────
Node A: write(stock = 50)

At this moment:
  Node A → returns 50  ✅
  Node B → returns 47  (hasn't caught up yet)
  Node C → returns 50  ✅

2 seconds later, all nodes → return 50  ✅
──────────────────────────────────────────────────────
```

**Conflicts:** What if two nodes each accept a different write before syncing?

```
Node A: write(x = "Alice")   ← User1 updates name
Node B: write(x = "Alicia")  ← User2 updates the same name concurrently

When they sync → CONFLICT.
```

Resolution strategies:
- **Last Write Wins (LWW):** Use timestamp — higher timestamp wins. Simple but loses data.
- **Multi-Value (siblings):** Store both, surface conflict to application. DynamoDB does this.
- **CRDTs:** Data structures that merge automatically without conflicts (counters, sets).

**Real systems:**
```
Cassandra     → Eventual by default (LWW resolution)
DynamoDB      → Eventual by default
DNS           → Classic eventual consistency example
Redis Cluster → Eventual (async replication between nodes)
Elasticsearch → Eventual (index propagation is async)
```

**ShopSphere example:**
```
product-service writes updated price to primary.
search-service reads from Elasticsearch index.

Elasticsearch index is eventually consistent with PostgreSQL.
→ Search results might show old price for a few seconds.
→ That's FINE for search — you don't need perfect freshness.
→ But payment must read from PostgreSQL directly (strong consistency).
```

---

## Side-by-Side Summary

```
Model              | Who sees the write immediately | Ordering guarantee    | Cost
───────────────────────────────────────────────────────────────────────────────────
Linearizable       | Everyone                       | Real-time global      | High latency
Sequential         | Everyone (agreed order)        | Logical global        | Medium latency
Causal             | Causally related readers       | Causal order only     | Medium
Read-Your-Writes   | Only the writer                | Per-session only      | Low
Eventual           | Nobody guaranteed              | None                  | Lowest (fastest)
```

---

## Which System Offers What

```
System          Default Consistency     Strongest Available
──────────────────────────────────────────────────────────
PostgreSQL      Strong (single node)    Linearizable
MySQL           Strong (single node)    Linearizable
MongoDB         Eventual (replica set)  Linearizable (majority read concern)
Cassandra       Eventual                Strong (QUORUM read + write, same DC)
DynamoDB        Eventual                Strong (ConsistentRead=true)
Redis           Eventual (cluster)      Strong (single node, no cluster)
Elasticsearch   Eventual                N/A — designed for search, not transactions
etcd / ZooKeeper  Strong               Linearizable
Kafka           Strong ordering (per partition)   Linearizable per partition
```

---

## The Practical Decision Framework

```
Ask yourself: "What's the worst case if a user reads stale data?"

CATASTROPHIC (oversell, double charge, lost data)
  → Strong consistency required
  → PostgreSQL with synchronous replication
  → etcd for distributed coordination

NOTICEABLE BUT RECOVERABLE (stale cart, slightly old price)
  → Read-your-writes or causal
  → DynamoDB session consistency
  → MongoDB with causal sessions

INVISIBLE TO USER (search index, recommendation feed, analytics)
  → Eventual consistency is perfect
  → Elasticsearch, Cassandra, Redis cache
```

**ShopSphere consistency map:**

```
Service              Store          Consistency Needed       Why
────────────────────────────────────────────────────────────────────────
order-service        PostgreSQL     Strong                   Can't oversell
payment-service      PostgreSQL     Strong                   Can't double charge
inventory-service    PostgreSQL     Strong                   Stock accuracy
user-service         PostgreSQL     Read-Your-Writes         User sees own updates
product-service      PostgreSQL     Eventual OK for reads    Price reads can lag slightly
search-service       Elasticsearch  Eventual                 Search index lag is fine
notification         RabbitMQ/Kafka Eventual                 Email a few seconds late is OK
Redis cache          Redis          Eventual                 Cache is a performance layer
```

---

## Interview Angles 🎯

**Q: What's the difference between strong consistency and linearizability?**
> Linearizability IS strong consistency — the strongest form. It means operations appear instantaneous and in real-time order across all nodes. "Strong consistency" is sometimes used loosely to mean "not eventual," but linearizability is the precise definition.

**Q: Cassandra says it can do strong consistency — how?**
> With QUORUM reads and writes — if write quorum + read quorum > total replicas, the read is guaranteed to overlap with the write. For 3 replicas: write to 2, read from 2 → at least 1 node has the latest data. More in the Quorum topic.

**Q: A user changes their password but logs in again and the old password works. What consistency issue is this?**
> Lack of read-your-writes consistency. The read after the write hit a replica that hadn't synced yet. Fix: route auth reads to primary for the session, or use sticky replica routing.

**Q: "Eventual consistency is fine for our use case" — how do you validate that claim?**
> Ask: what's the maximum acceptable staleness window? What happens on a conflict — who wins? Is the operation idempotent? If two nodes accept conflicting writes, what's the merge strategy? "Eventual" isn't a free pass — you need to reason about conflict resolution.

---

Say **next** when you're ready for **Topic 3: Lamport Logical Clocks** 🚀
