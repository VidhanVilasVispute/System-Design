# Stage 7 — Distributed Systems
## Topic 5: Consensus Algorithms (Raft & Paxos)

---

## The Problem We're Solving

You have 5 nodes. They all need to **agree on the same value** — who the leader is, what the next log entry is, whether a transaction committed.

Simple right? Just have them vote.

But here's the catch:

```
- Any node can crash at any time
- Messages can be delayed, duplicated, or dropped
- You can't tell if a node crashed or is just slow
- A crashed node might come back with stale state
- Two nodes might both think they're the leader
```

This is the **consensus problem** — and it's provably hard.

**FLP Impossibility (1985):** In a fully asynchronous system, it is **impossible** to guarantee consensus if even one node can fail.

The way out: make timing assumptions. If the network is *eventually* reliable, consensus is achievable. That's the real world — and that's what Raft and Paxos solve.

---

## What Consensus Gives You

A consensus algorithm guarantees:

```
Agreement:   All non-faulty nodes decide the same value
Validity:    The decided value was proposed by some node
Termination: All non-faulty nodes eventually decide

These three together = consensus
```

**Where consensus is used:**

```
etcd         → Leader election + key-value log (Kubernetes uses this)
ZooKeeper    → Distributed coordination (Kafka used it pre-KRaft)
CockroachDB  → Multi-Raft for distributed transactions
Kafka KRaft  → Replaced ZooKeeper with native Raft
Your own DB  → Primary election in PostgreSQL streaming replication
```

---

## Paxos — The Original (and Notoriously Hard to Understand)

Paxos was introduced by Leslie Lamport in 1989 (same person as Lamport clocks). He submitted the paper, reviewers said it was too confusing, it sat unpublished until 1998.

Even Lamport admitted:

> *"The dirty secret of the NSDI community is that at most five people really understand Paxos."*

Let's understand the core idea without getting lost in edge cases.

---

### Paxos Roles

```
Proposer   → Wants to get a value agreed upon
Acceptor   → Votes on proposals (usually the same nodes as proposers)
Learner    → Learns the decided value (clients, downstream services)
```

In practice — every node plays all three roles.

---

### Paxos in Two Phases

**Phase 1 — Prepare / Promise**

```
Proposer picks a proposal number N (must be unique, higher than any seen)

Proposer → sends Prepare(N) to majority of Acceptors

Each Acceptor responds:
  IF N > any proposal number seen before:
    → Promise to never accept proposals < N
    → Return (highest accepted proposal, its value) if any

Proposer collects promises from majority → proceeds to Phase 2
```

**Phase 2 — Accept / Accepted**

```
Proposer picks value V:
  → If any acceptor returned a previously accepted value → must use that value
  → Otherwise → free to propose any value

Proposer → sends Accept(N, V) to majority of Acceptors

Each Acceptor:
  IF N >= highest promised number:
    → Accept the proposal
    → Notify Learners

If majority accept → VALUE IS DECIDED ✅
```

---

### Paxos Walkthrough

```
5 nodes: A, B, C, D, E
Proposer = A, wants to commit value "X"

─────────────────────────────────────────────────────────────────
Phase 1:
  A sends Prepare(N=5) to B, C, D, E

  B: "Haven't seen N>5, I promise. No prior accepted value."
  C: "Haven't seen N>5, I promise. No prior accepted value."
  D: (crashed — no response)
  E: "Haven't seen N>5, I promise. No prior accepted value."

  A has 3 promises (majority of 5) → proceeds ✅

─────────────────────────────────────────────────────────────────
Phase 2:
  No prior accepted values returned → A can use its own value "X"
  A sends Accept(N=5, V="X") to B, C, E

  B: accepts → notifies learners
  C: accepts → notifies learners
  E: accepts → notifies learners

  Majority accepted → "X" IS DECIDED ✅
─────────────────────────────────────────────────────────────────
```

D was down the whole time. When D comes back, it learns the decided value from others. The system made progress with 4/5 nodes. ✅

---

### The Classic Paxos Problem — Dueling Proposers

```
Two proposers A and B keep interrupting each other:

A sends Prepare(N=1) → gets promises
B sends Prepare(N=2) → invalidates A's round
A sends Prepare(N=3) → invalidates B's round
B sends Prepare(N=4) → invalidates A's round
... forever

This is called the "livelock" problem in Paxos.
Solution: elect ONE distinguished proposer (leader) — Multi-Paxos.
```

Multi-Paxos skips Phase 1 once a stable leader is elected — Phase 2 only for each log entry. This is what systems actually implement. But the algorithm is complex to get right.

**This is why Raft was invented.**

---

## Raft — Paxos Made Understandable

Raft was designed in 2013 by Diego Ongaro with one explicit goal:

> *"Raft is a consensus algorithm designed to be more understandable than Paxos."*

His PhD dissertation is literally titled "Consensus: Bridging Theory and Practice."

Raft decomposes consensus into three independent subproblems:

```
1. Leader Election    → pick one leader, everyone follows it
2. Log Replication    → leader receives writes, replicates to followers
3. Safety             → if a log entry is committed, it will always be there
```

---

## Raft Core Concepts

### Terms

Raft divides time into **terms** — monotonically increasing integers.

```
Term 1: Node A is leader
        │
        ▼ A crashes
Term 2: New election → Node B wins
        │
        ▼ B crashes
Term 3: New election → Node C wins
```

Each term begins with an election. If election succeeds → one leader for that term. If no majority → term ends with no leader, new term begins.

Terms are logical clocks for leadership. A node that wakes up from a crash sees a higher term → knows it missed something → updates itself.

---

### Node States

```
Every Raft node is in exactly one of three states:

┌─────────────┐         election timeout         ┌─────────────┐
│  FOLLOWER   │ ──────────────────────────────▶  │  CANDIDATE  │
│             │                                   │             │
│ Receives    │ ◀──────────────────────────────   │ Requests    │
│ heartbeats  │         loses election            │ votes       │
└─────────────┘                                   └──────┬──────┘
       ▲                                                  │
       │                                        wins majority
       │                                                  │
       │            steps down (higher term)              ▼
       └─────────────────────────────────── ┌─────────────┐
                                            │   LEADER    │
                                            │             │
                                            │ Sends       │
                                            │ heartbeats  │
                                            │ Replicates  │
                                            │ log entries │
                                            └─────────────┘
```

---

## Raft: Leader Election

```
Normal operation:
  Leader sends heartbeat (AppendEntries with no entries) to all followers
  every ~150ms

  Followers reset their election timeout on each heartbeat received.
  Election timeout = random between 150ms–300ms (randomness prevents ties)

If a follower doesn't hear from leader within timeout:
  → Assumes leader is dead
  → Increments term
  → Transitions to CANDIDATE
  → Votes for itself
  → Sends RequestVote to all other nodes
```

**Voting rules:**

```
A node grants a vote IF:
  1. It hasn't voted in this term yet
  2. The candidate's log is at least as up-to-date as the voter's log
     (prevents stale nodes from becoming leader)

A candidate wins if it gets votes from MAJORITY (n/2 + 1) of nodes
```

**Why majority?** Because two majorities always overlap by at least one node. That node acts as the link — it can only vote for one candidate, so only one can win.

```
5 nodes: need 3 votes
  Candidate A gets: [A, B, C] → wins ✅
  Candidate B gets: [B, D, E] → impossible simultaneously
                                C is in A's majority, can't be in B's ✅
```

---

## Raft: Log Replication

Once a leader is elected — all writes go through it:

```
Step 1: Client sends write to Leader
        "Set x = 10"

Step 2: Leader appends to its OWN log (not committed yet)
        Log: [..., entry(term=3, index=47, cmd="set x=10")]

Step 3: Leader sends AppendEntries to ALL followers in parallel

Step 4: Followers append to their logs, respond OK

Step 5: Once MAJORITY responds OK:
        → Entry is COMMITTED
        → Leader applies to state machine
        → Leader responds to client ✅

Step 6: Next heartbeat tells followers the entry is committed
        → Followers apply to their state machines
```

Visualized:

```
         Leader          Follower B       Follower C
           │                 │                │
Client ──▶ │ append(47)      │                │
           │ ──AppendEntries(47)──────────────▶│
           │ ──AppendEntries(47)──▶│           │
           │                 │ OK  │           │ OK
           │ ◀───────────────┘     │ ◀─────────┘
           │ (majority = committed)
           │ apply to state machine
Client ◀── │ ✅ success
           │ ...heartbeat... tells followers to commit
```

---

## Raft: Safety — The Log Matching Property

Raft guarantees:

```
If two log entries have the same index and term → they are identical
If two logs agree at index N → they agree on all entries before N
```

This means: once an entry is committed, **it will never be overwritten** — even if the leader crashes.

**How?** The leader consistency check:
- Each AppendEntries includes the index and term of the entry just before the new one
- Followers reject if their log doesn't match at that point
- Leader must find where logs diverge and send all missing entries

---

## Raft vs Paxos — Side by Side

```
Property              Paxos                    Raft
──────────────────────────────────────────────────────────────
Understandability     Complex, many edge cases  Designed to be simple
Leader               Multi-Paxos needs one     Explicit, first-class concept
Log structure         Not specified             Explicit append-only log
Term concept          Ballot numbers            Explicit terms
Membership changes    Not specified             Joint consensus / config changes
Real implementations  Chubby (Google)           etcd, CockroachDB, Kafka KRaft
PhD dissertations     None (just papers)        Yes — literally built for clarity
```

---

## Fault Tolerance — How Many Failures Can You Handle?

```
Cluster size    Failures tolerated    Majority needed
─────────────────────────────────────────────────────
1               0                     1
3               1                     2
5               2                     3
7               3                     4

Formula: can tolerate F failures with 2F+1 nodes
```

**Why odd numbers?** Even numbers don't help:

```
4 nodes, 1 failure:
  Remaining: 3 nodes
  Majority of 4 = 3 → still need 3 → can only tolerate 1 failure
  (same as 3 nodes!)

4 nodes, 2 failures:
  Remaining: 2 nodes
  Can't reach majority of 3 → cluster unavailable
  
So 4 nodes = same fault tolerance as 3, with more cost.
```

---

## Where This Lives in ShopSphere

ShopSphere doesn't implement Raft internally — but it **relies on systems that do**:

```
etcd (via Kubernetes):
─────────────────────────────────────────────────────────────
All ShopSphere service discovery, ConfigMaps, Secrets → stored in etcd
etcd uses Raft internally → 3 or 5 etcd nodes in production cluster
If etcd leader crashes → Raft elects new leader in ~300ms
Your pods keep running; new pod scheduling waits until etcd recovers

Kafka KRaft (post Kafka 3.3):
─────────────────────────────────────────────────────────────
ShopSphere's Kafka uses KRaft mode (no ZooKeeper)
Kafka controllers use Raft for partition leadership decisions
order-events, payment-events topics → partition leaders elected via Raft

PostgreSQL (streaming replication):
─────────────────────────────────────────────────────────────
Not pure Raft, but similar:
  Primary receives writes → streams WAL to standbys
  Patroni (common HA tool) uses etcd/Consul to do leader election
  If primary dies → Patroni uses distributed consensus to elect new primary
─────────────────────────────────────────────────────────────
```

---

## The Raft Guarantee That Matters Most

```
As long as a MAJORITY of nodes are alive and can communicate:
  → The cluster makes progress
  → Committed entries are never lost
  → Exactly one leader exists at any time

If majority is unavailable (network partition, too many crashes):
  → Cluster stops accepting writes (CP behavior)
  → It will NOT serve stale reads (unlike eventual consistent systems)
  → Recovers automatically when majority is restored
```

This is why etcd and ZooKeeper are CP systems — they choose consistency over availability.

---

## Interview Angles 🎯

**Q: What is the consensus problem?**
> Getting a set of distributed nodes to agree on a single value, even when some nodes crash or messages are lost. It must satisfy agreement (all decide same value), validity (decided value was proposed), and termination (all nodes eventually decide).

**Q: Why was Raft invented if Paxos already existed?**
> Paxos is notoriously hard to understand and leaves many practical details unspecified — log structure, membership changes, leader election. Raft was designed from the ground up for understandability, with explicit leader election, a well-defined append-only log, and clear term semantics. It's easier to implement correctly.

**Q: How does Raft handle a leader crash?**
> Followers stop receiving heartbeats, their election timeout fires, they become candidates and request votes. A node with the most up-to-date log wins the election. The new leader then replicates any uncommitted entries to followers. Committed entries are never lost because the winner must have seen them (log completeness guarantee).

**Q: What's the difference between a committed and an applied log entry in Raft?**
> Committed means a majority of nodes have the entry in their logs — it will never be lost. Applied means the entry has been executed against the state machine (e.g., the write has been applied to the DB). Leaders apply immediately after committing; followers apply after being told the entry is committed via the next heartbeat.

**Q: Why do Raft clusters use odd numbers of nodes?**
> Because even numbers provide no additional fault tolerance. A 4-node cluster needs a majority of 3, same as a 3-node cluster. The extra node adds cost without improving the number of tolerable failures. Formula: 2F+1 nodes to tolerate F failures.

**Q: etcd is used by Kubernetes — what does that mean for your microservices?**
> Every piece of Kubernetes state — pod specs, ConfigMaps, Secrets, service endpoints — is stored in etcd. etcd uses Raft internally, so it's always consistent. If the etcd cluster loses majority quorum, Kubernetes can't schedule new pods or update configs, though running pods continue unaffected.

---

Say **next** for **Topic 6: Leader Election** — picking one coordinator without a central authority 🚀
