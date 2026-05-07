# Stage 7 — Distributed Systems
## Topic 4: Vector Clocks

---

## The Problem Lamport Clocks Left Unsolved

From Topic 3 — Lamport clocks guarantee:

```
A → B  ⟹  C(A) < C(B)   ✅

But NOT the reverse:
C(A) < C(B)  ⟹  A → B   ❌ (might be concurrent)
```

You cannot look at two Lamport timestamps and know whether one event **caused** the other or whether they happened **independently on different nodes**.

This matters enormously in practice:

```
ShopSphere — Two users update the same product description concurrently:

Node A: description = "Red Shoes, size 10"   (Lamport ts = 5)
Node B: description = "Red Shoes, size 11"   (Lamport ts = 7)

Lamport says: B has higher timestamp → B wins (Last Write Wins)

But were they actually concurrent? Did one user see the other's write first?
Lamport cannot answer this. We might be silently discarding a valid update.
```

**Vector Clocks** solve this by tracking **per-node knowledge** — not a single global counter, but a vector of counters, one per node.

---

## The Core Idea

Instead of one integer, each node maintains a **vector** (array) of counters — one slot per node in the system.

```
3-node system: Nodes A, B, C

Each node's vector clock:
  Node A maintains: [A=0, B=0, C=0]
  Node B maintains: [A=0, B=0, C=0]
  Node C maintains: [A=0, B=0, C=0]

"My slot = how many events I've processed"
"Other slots = how many events from that node I've seen"
```

Think of it as each node tracking:
> *"Here's everything I know happened, and where I learned it from."*

---

## The Algorithm — Three Rules

```
Rule 1 — Local event on Node X:
  Increment X's own slot
  VC[X] = VC[X] + 1

Rule 2 — Send message from Node X:
  Increment own slot first
  VC[X] = VC[X] + 1
  Attach entire vector to message

Rule 3 — Receive message on Node X with vector VM:
  For each slot i:
    VC[i] = max(VC[i], VM[i])
  Then increment own slot:
    VC[X] = VC[X] + 1
```

The `max` merge means: *"I now know about everything the sender knew, plus my own history."*

---

## Step-by-Step Walkthrough

Three nodes: **A**, **B**, **C** — notation: `[A, B, C]`

```
Initial state:
  A: [0,0,0]    B: [0,0,0]    C: [0,0,0]

─────────────────────────────────────────────────────────────────
Event 1: A does local event
  A increments own slot
  A: [1,0,0]    B: [0,0,0]    C: [0,0,0]

─────────────────────────────────────────────────────────────────
Event 2: A sends message to B  (A increments first, then sends)
  A: [2,0,0] ──── msg([2,0,0]) ────▶ B

Event 3: B receives from A
  B merges: max([0,0,0], [2,0,0]) = [2,0,0]
  B increments own slot
  B: [2,1,0]    A: [2,0,0]    C: [0,0,0]

─────────────────────────────────────────────────────────────────
Event 4: B sends message to C
  B: [2,2,0] ──── msg([2,2,0]) ────▶ C

Event 5: C receives from B
  C merges: max([0,0,0], [2,2,0]) = [2,2,0]
  C increments own slot
  C: [2,2,1]

─────────────────────────────────────────────────────────────────
Event 6: A does another local event (INDEPENDENT — no messages)
  A: [3,0,0]    ← A has NOT seen B or C's events
```

Now the state is:

```
A: [3,0,0]   ← A knows only its own 3 events
B: [2,2,0]   ← B has seen A's first 2 events, its own 2
C: [2,2,1]   ← C has seen everything B knew, plus itself
```

---

## Comparing Vector Clocks — The Comparison Rules

Given two vector clocks **V1** and **V2**:

```
V1 = V2  (identical)
  → Same event

V1 < V2  (V1 happened-before V2)
  → All slots of V1 ≤ corresponding slots of V2
  → At least one slot of V1 < V2

V1 > V2  (V2 happened-before V1)
  → Reverse of above

V1 ∥ V2  (CONCURRENT — this is what Lamport can't detect!)
  → V1[i] > V2[i] for some i
  → V1[j] < V2[j] for some j
  → Neither dominates the other
```

Let's apply this:

```
A's event:  VA = [3,0,0]
C's event:  VC = [2,2,1]

Compare slot by slot:
  A slot: VA[A]=3  vs  VC[A]=2  → VA wins here
  B slot: VA[B]=0  vs  VC[B]=2  → VC wins here
  C slot: VA[C]=0  vs  VC[C]=1  → VC wins here

Mixed result → CONCURRENT ∥

Meaning: A's event at [3,0,0] and C's event at [2,2,1]
         happened independently. Neither caused the other.
```

This is **exactly** what Lamport clocks couldn't tell you.

---

## The Real Power — Detecting Conflicting Writes

```
ShopSphere — product-service running on 2 nodes:

Initial state: product description = "Blue Sneakers"
Both nodes start at: [A=1, B=1]

─────────────────────────────────────────────────────────────────
User 1 updates via Node A:
  description = "Blue Sneakers, size 9"
  Vector clock: [A=2, B=1]

User 2 updates via Node B (concurrently, hasn't seen A's write):
  description = "Blue Sneakers, waterproof"
  Vector clock: [A=1, B=2]
─────────────────────────────────────────────────────────────────

Now nodes sync. Compare:
  [A=2, B=1] vs [A=1, B=2]
  → A wins slot A, B wins slot B → CONCURRENT CONFLICT ∥

System knows: these are TWO valid writes that happened independently.
It should NOT silently discard one.

Options:
  1. Surface conflict to user ("Which version do you want?")
  2. Merge automatically if possible ("Blue Sneakers, size 9, waterproof")
  3. Apply business rule (latest user-facing write wins)
─────────────────────────────────────────────────────────────────
```

Compare this to **Lamport LWW** which would have silently thrown one away.

---

## How DynamoDB Uses Vector Clocks

DynamoDB calls them **"version vectors"** and uses them to detect **siblings** — conflicting versions of the same item.

```
DynamoDB flow:
────────────────────────────────────────────────────────────────
1. Client reads item → gets version vector [A=3, B=3]
2. Client modifies item
3. Client writes back with the read version vector as context

DynamoDB checks:
  If stored vector ≤ submitted vector → safe update, no conflict
  If stored vector ∥ submitted vector → CONFLICT → creates sibling

Application must resolve siblings before next write.
────────────────────────────────────────────────────────────────
```

Amazon's Dynamo paper (2007) describes this in detail — foundational reading for distributed systems.

---

## How Riak Uses Vector Clocks

Riak (distributed key-value store) stores **siblings** explicitly and lets the application merge them:

```
Client reads key "cart:user42":
  → Gets two siblings:
     [A=2, B=1]: {items: ["shoes"]}
     [A=1, B=2]: {items: ["shoes", "jacket"]}

Application merges: union of item sets = {shoes, jacket}
Writes back merged value with merged vector clock.
```

This is how **shopping carts in distributed systems** work reliably — you never silently lose an item the user added from a different device.

---

## Vector Clocks in ShopSphere

```
review-service — two users edit the same review concurrently
(yes, unusual, but imagine a collaborative editing feature)

Node 1 starts: [N1=1, N2=1]

User A edits via Node 1: "Great quality!"       → [N1=2, N2=1]
User B edits via Node 2: "Great quality, fast!" → [N1=1, N2=2]

Sync:
  [N1=2, N2=1] ∥ [N1=1, N2=2] → CONFLICT detected

review-service can:
  → Keep both versions (show both to admin for resolution)
  → Apply merge strategy (longer text wins)
  → Surface to original author to pick
```

For **inventory-service** — concurrent writes to stock levels are even more critical:

```
Node 1: stock 10 → 8  (sold 2)   [N1=2, N2=1]
Node 2: stock 10 → 9  (sold 1)   [N1=1, N2=2]

Concurrent conflict detected!
Correct merge: stock = 10 - 2 - 1 = 7  (not pick one, merge the deltas)
Wrong merge (LWW): stock = 9 → you just gave away free inventory
```

---

## Lamport vs Vector Clocks — Side by Side

```
Property                    Lamport Clock    Vector Clock
──────────────────────────────────────────────────────────
Data structure              Single integer   Array of integers (one per node)
Space per event             O(1)             O(n) where n = number of nodes
Detects happened-before     ✅ Yes           ✅ Yes
Detects concurrency         ❌ No            ✅ Yes
Causal ordering             Partial          Full
Used in                     Kafka offsets,   DynamoDB, Riak,
                            etcd, debugging  CRDTs, Git
```

---

## Wait — What About Git?

Git commits are essentially vector clocks.

```
Every commit stores:
  - Its own hash (identity)
  - Parent commit hash(es)

Two branches that diverged from the same commit:

        main: A → B → C
                    ↘
        feature:    D → E

C and E are CONCURRENT — neither is an ancestor of the other.
Git detects this as a MERGE CONFLICT.

That's vector clock concurrency detection — just with commit hashes
instead of integer vectors.
```

---

## Implementation Sketch (Java)

```java
public class VectorClock {
    private final Map<String, Long> clock = new ConcurrentHashMap<>();
    private final String nodeId;

    public VectorClock(String nodeId) {
        this.nodeId = nodeId;
    }

    // Local event or before sending
    public Map<String, Long> tick() {
        clock.merge(nodeId, 1L, Long::sum);
        return Collections.unmodifiableMap(clock);
    }

    // On receive — merge then tick
    public Map<String, Long> receive(Map<String, Long> remote) {
        remote.forEach((node, ts) ->
            clock.merge(node, ts, Math::max)   // take max per slot
        );
        clock.merge(nodeId, 1L, Long::sum);    // increment own slot
        return Collections.unmodifiableMap(clock);
    }

    // Compare two vector clocks
    public enum Relation { BEFORE, AFTER, CONCURRENT, EQUAL }

    public static Relation compare(Map<String, Long> v1, Map<String, Long> v2) {
        Set<String> allNodes = new HashSet<>();
        allNodes.addAll(v1.keySet());
        allNodes.addAll(v2.keySet());

        boolean v1Greater = false, v2Greater = false;

        for (String node : allNodes) {
            long t1 = v1.getOrDefault(node, 0L);
            long t2 = v2.getOrDefault(node, 0L);
            if (t1 > t2) v1Greater = true;
            if (t2 > t1) v2Greater = true;
        }

        if (v1Greater && v2Greater) return Relation.CONCURRENT;
        if (!v1Greater && !v2Greater) return Relation.EQUAL;
        return v1Greater ? Relation.AFTER : Relation.BEFORE;
    }
}
```

---

## The Mental Model — Final Summary

```
Lamport Clock:
  "I can tell you event A came before event B
   if A's number is lower.
   But I can't tell you if they're concurrent."

Vector Clock:
  "I can tell you EXACTLY the relationship between any two events:
   - A caused B
   - B caused A
   - They happened independently (concurrent)
   No ambiguity."

The cost: instead of 1 number, you store N numbers (one per node).
For systems with thousands of nodes, this is expensive.
That's why DynamoDB uses pruned/bounded version vectors in practice.
```

---

## Interview Angles 🎯

**Q: What does a vector clock tell you that a Lamport clock doesn't?**
> Whether two events are concurrent. With a Lamport clock, a lower timestamp just means "possibly before" — you can't distinguish causality from coincidence. Vector clocks track per-node knowledge, so if neither vector dominates the other, the events are definitively concurrent.

**Q: How does DynamoDB handle concurrent writes?**
> It uses version vectors (similar to vector clocks) to detect conflicting writes. When a conflict is detected, DynamoDB stores both versions as "siblings" and surfaces them to the application, which must resolve the conflict on the next read.

**Q: Why does Git show a merge conflict?**
> Because both branches have commits that are concurrent — neither is an ancestor of the other. Git's DAG structure is essentially a vector clock — if two commits have no direct ancestry relationship, they're concurrent writes to the same history and must be merged.

**Q: What's the scalability problem with vector clocks?**
> The vector size grows with the number of nodes — O(n) per event. In a system with thousands of nodes, this becomes expensive. Practical systems use bounded version vectors, pruning old entries, or alternative structures like dotted version vectors.

**Q: In a shopping cart, why is union-merge the right strategy for concurrent writes?**
> Because adding an item to a cart is a user intent that should never be silently lost. If two devices both added different items concurrently, the correct result is both items in the cart. Union merge (CRDT set semantics) captures this correctly. LWW would silently drop one user's action.

---

Say **next** for **Topic 5: Consensus Algorithms (Raft & Paxos)** — how nodes agree on a value when some may crash 🚀
