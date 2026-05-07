# Stage 7 — Distributed Systems
## Topic 8: Two-Phase Commit (2PC)

---

## The Problem We're Solving

In a single database, transactions are simple:

```java
BEGIN;
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 42;
  INSERT INTO orders (product_id, user_id, status) VALUES (42, 7, 'PLACED');
COMMIT;  // both happen, or neither happens
```

The database guarantees atomicity — **all or nothing**.

Now in ShopSphere — placing an order touches **three separate services**, each with its own database:

```
order-service    → INSERT into orders DB
inventory-service → UPDATE stock in inventory DB
payment-service  → INSERT charge in payment DB

All three must succeed together.
Or ALL must roll back.

There is no single COMMIT that spans all three.
That's the distributed transaction problem.
```

**Two-Phase Commit (2PC)** is the classical solution. It's worth understanding deeply — because almost every modern system explicitly chooses NOT to use it, and you need to know exactly why.

---

## The Core Idea

2PC introduces a **coordinator** — a node that orchestrates the transaction across all **participants** (the individual services/databases).

```
Coordinator: order-service (or a dedicated transaction manager)
Participants: inventory-service, payment-service, order-service DB

Two phases:
  Phase 1 — Prepare:  "Can you commit? Lock your resources."
  Phase 2 — Commit:   "Everyone said yes → now actually commit."
                   OR "Someone said no → everyone roll back."
```

The insight: separate the **"can you?"** from the **"do it."**

---

## Phase 1 — The Prepare Phase

```
Coordinator sends PREPARE to all participants.

Each participant:
  1. Writes the transaction to its WAL (Write-Ahead Log)
  2. Acquires all necessary locks on affected rows
  3. Verifies the transaction can succeed
  4. Responds:
       YES  → "I can commit, I've locked resources, I'm ready"
       NO   → "I cannot commit (constraint violation, timeout, etc.)"
  
Crucially: After responding YES, a participant is STUCK.
  It has locked rows and CANNOT release them until the coordinator decides.
```

```
                    Coordinator
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
    inventory-svc   payment-svc   order-svc
    
    PREPARE ────▶   PREPARE ────▶  PREPARE ────▶
    
    ◀──── YES       ◀──── YES      ◀──── YES

    (All rows locked. All participants waiting.)
```

---

## Phase 2 — The Commit Phase

**If ALL participants voted YES:**

```
Coordinator writes COMMIT to its own WAL
Coordinator sends COMMIT to all participants

Each participant:
  → Applies the transaction
  → Releases locks
  → Sends ACK to coordinator
  → Done ✅
```

**If ANY participant voted NO:**

```
Coordinator sends ROLLBACK to all participants

Each participant:
  → Undoes any changes
  → Releases locks
  → Sends ACK
  → Done ✅
```

---

## Full Walkthrough — Happy Path

```
ShopSphere order placement:

─────────────────────────────────────────────────────────────────
[Phase 1 — Prepare]

Coordinator → inventory-service:  PREPARE (decrement stock by 1)
Coordinator → payment-service:    PREPARE (charge $99)
Coordinator → order-service DB:   PREPARE (insert order record)

inventory-service:  checks stock > 0 ✅, locks row, responds YES
payment-service:    checks card valid ✅, locks charge, responds YES
order-service DB:   validates schema ✅, responds YES

Coordinator: received YES from all 3 ✅

─────────────────────────────────────────────────────────────────
[Phase 2 — Commit]

Coordinator writes "COMMIT" to its own WAL first
Coordinator → all participants: COMMIT

inventory-service:  decrements stock, releases lock, ACKs
payment-service:    charges card, releases lock, ACKs
order-service DB:   inserts order, ACKs

Transaction complete. ✅
─────────────────────────────────────────────────────────────────
```

---

## The Blocking Problem — Why 2PC Is Dangerous

Here's where 2PC falls apart. Let's look at what happens when things go wrong.

### Scenario 1: Participant Crashes After Voting YES

```
[Phase 1 complete — all voted YES]
Coordinator sends COMMIT

inventory-service: commits ✅
payment-service:   CRASHES 💥  ← never receives COMMIT
order-service DB:  commits ✅

State of the world:
  inventory decremented ✅
  payment NOT charged   ❌
  order record inserted ✅

payment-service restarts.
It knows it voted YES (from its WAL).
But it doesn't know if coordinator sent COMMIT or ROLLBACK.

It must ASK the coordinator: "What should I do?"
Until coordinator responds → payment-service HOLDS ITS LOCKS.
```

### Scenario 2: Coordinator Crashes After Receiving All YES Votes

```
Coordinator receives YES from all participants.
Coordinator writes COMMIT to WAL.
Coordinator CRASHES 💥 before sending COMMIT to participants.

All three participants are now:
  → Holding locks on their rows
  → Waiting for COMMIT or ROLLBACK
  → Cannot proceed
  → Cannot time out (they voted YES — they're committed to waiting)

This is the BLOCKING problem.
The entire transaction is frozen until coordinator recovers.
```

```
Time passing...
─────────────────────────────────────────────────────────────────

inventory-service:  stock row LOCKED  ⏳  (other orders can't update it)
payment-service:    charge LOCKED     ⏳
order-service DB:   order row LOCKED  ⏳

Users trying to buy the same product: all blocked.
New orders for affected products: impossible.

Coordinator is down for 10 minutes → 10 minutes of frozen inventory.
─────────────────────────────────────────────────────────────────
```

This is called **the blocking problem** — 2PC is a **blocking protocol**. In the presence of failures, participants have no safe way to unilaterally decide. They must wait.

---

## Why Participants Can't Just Time Out

You might think: "Just let participants time out and roll back if coordinator doesn't respond."

The problem:

```
Participant times out → decides to ROLLBACK

But coordinator had actually sent COMMIT to other participants
and they already committed!

Result:
  inventory-service: ROLLED BACK (stock not decremented)
  payment-service:   COMMITTED   (customer charged)
  order-service:     COMMITTED   (order exists)

Customer was charged but inventory wasn't decremented.
Even worse state than before. 💥
```

Once a participant votes YES, it has surrendered its autonomy. It **must** wait for the coordinator's decision. That's the fundamental trap of 2PC.

---

## The Coordinator's WAL Is Critical

Notice this step earlier:

```
Coordinator writes "COMMIT" to its own WAL FIRST
THEN sends COMMIT to participants
```

This is deliberate. When the coordinator recovers from a crash:

```
If WAL says COMMIT was written → send COMMIT to all participants
If WAL says nothing about COMMIT → send ROLLBACK to all participants

The WAL is the source of truth for what the coordinator decided.
This is the only safe recovery path.
```

---

## 2PC Failure Matrix

```
Failure Point                    Result
──────────────────────────────────────────────────────────────────────
Participant fails before PREPARE  Coordinator times out → ROLLBACK ✅
Participant fails after YES vote  Blocks until coordinator recovers
Coordinator fails before COMMIT   All participants block ← worst case
Coordinator fails after COMMIT    Some committed, some waiting → blocks
Network partition                 Participants can't reach coordinator → blocks
Coordinator recovers              Reads WAL → sends decision → unblocks ✅
```

---

## Three-Phase Commit (3PC) — The Attempted Fix

3PC adds a **pre-commit phase** to eliminate blocking:

```
Phase 1 — Prepare:      same as 2PC
Phase 2 — Pre-Commit:   coordinator tells everyone "I'm going to commit"
                        participants ACK, now everyone KNOWS the decision
Phase 3 — Commit:       actual commit

If coordinator crashes after Phase 2:
  Any participant can take over and commit
  (they all know the decision was COMMIT)
```

**Why 3PC is still not used:**

```
1. Extra round trip → more latency than 2PC
2. Still not safe under network partitions
   (pre-commit message might reach some nodes, not others)
3. More complex to implement correctly
4. Modern alternatives (Saga) are better tradeoffs

3PC exists mostly in textbooks. Real systems use Saga.
```

---

## Where 2PC IS Actually Used

Despite its problems, 2PC is used when you truly need ACID across nodes and can accept the blocking risk:

```
XA Transactions (Java EE / Jakarta EE):
  Standard API for 2PC across multiple resources
  JTA (Java Transaction API) coordinates:
    → Database (via XA JDBC driver)
    → JMS queue (via XA connection factory)

  order-service XA transaction:
    BEGIN XA
      INSERT into orders DB
      PUBLISH to JMS queue
    COMMIT XA
    → Either both happen or neither

Google Spanner:
  Uses 2PC internally but with TrueTime (GPS/atomic clocks)
  to bound clock uncertainty
  → Makes 2PC safe at global scale
  → Still has latency cost

CockroachDB:
  Uses 2PC internally for cross-range transactions
  with Raft for coordinator durability
  → Coordinator survives crashes via Raft log
  → Solves the coordinator failure problem

MySQL / PostgreSQL XA:
  Supported but rarely used in microservices
  Used in some banking/financial systems
  where atomicity is non-negotiable
```

---

## Why Microservices Avoid 2PC

```
Microservice Principle          2PC Violation
──────────────────────────────────────────────────────────────────
Services own their data         2PC requires cross-service locking
Services are independently      2PC creates tight coupling between
  deployable                      services during transactions
Services can fail independently 2PC coordinator = single point of failure
Network is unreliable           2PC blocks on network failures
High availability required      2PC can freeze parts of your system
```

The core conflict: **microservices are designed for independence**. 2PC creates the tightest possible coupling — one service's crash freezes every other service involved in the transaction.

---

## ShopSphere — What Happens Without 2PC?

ShopSphere doesn't use 2PC. So how does it handle multi-service consistency?

```
Current order flow (eventual consistency via events):

order-service:
  1. Save order locally (status = PENDING)     ← single local transaction
  2. Publish OrderCreated to Kafka              ← outbox pattern (Topic 12)

payment-service listens to OrderCreated:
  3. Charge payment
  4. Publish PaymentProcessed or PaymentFailed

inventory-service listens to PaymentProcessed:
  5. Decrement stock
  6. Publish InventoryDecremenated

order-service listens to all outcomes:
  7. Update order status to CONFIRMED or handle failures via Saga
```

No 2PC. No cross-service locks. Each step is a local transaction. Failures are handled by compensating transactions (the Saga pattern — Topic 9).

---

## Java XA Example (For Understanding, Not Recommendation)

```java
// What 2PC looks like in Java with JTA — educational only
// Spring Boot + Atomikos or Bitronix transaction manager

@Configuration
public class XAConfig {

    @Bean
    public UserTransaction userTransaction() throws SystemException {
        UserTransactionImp tx = new UserTransactionImp();
        tx.setTransactionTimeout(300);
        return tx;
    }

    @Bean
    public TransactionManager transactionManager() {
        return new UserTransactionManager();
        // Atomikos acts as the 2PC coordinator
    }
}

// Service layer
@Service
public class OrderService {

    @Autowired UserTransaction utx;
    @Autowired OrderRepository orderRepo;     // XA-capable DataSource
    @Autowired JmsTemplate jmsTemplate;      // XA-capable JMS

    public void placeOrder(Order order) throws Exception {
        utx.begin();  // start XA transaction
        try {
            orderRepo.save(order);           // participant 1
            jmsTemplate.send("orders",       // participant 2
                session -> session.createTextMessage(order.getId()));
            utx.commit();  // 2PC across DB + JMS
        } catch (Exception e) {
            utx.rollback();
            throw e;
        }
    }
}
```

The Atomikos transaction manager acts as the coordinator, running full 2PC between the database and the JMS broker. Both commit atomically or both roll back.

---

## 2PC vs Saga — The Core Tradeoff Preview

```
Property              2PC                      Saga (next topic)
──────────────────────────────────────────────────────────────────
Consistency           Strong (ACID)            Eventual (BASE)
Blocking              Yes — locks held         No — async steps
Failure handling      Coordinator decides      Compensating transactions
Coupling              Tight (sync locks)       Loose (async events)
Latency               High (2 round trips)     Low (async)
Throughput            Low (locking)            High (no locks)
Complexity            Protocol complexity      Business logic complexity
Used in               Financial DBs, Spanner   Microservices everywhere
```

2PC trades **availability and latency** for **strong consistency**.
Saga trades **strong consistency** for **availability and throughput**.

For ShopSphere — Saga is the right choice. We'll build it in detail next topic.

---

## Interview Angles 🎯

**Q: Explain 2PC and why it's called "two-phase."**
> 2PC has two phases. Phase 1 (Prepare): the coordinator asks all participants if they can commit — they lock resources and vote YES/NO. Phase 2 (Commit/Rollback): if all voted YES the coordinator sends COMMIT; if any voted NO it sends ROLLBACK. It's called two-phase because the decision-making (can you?) is separated from the execution (do it).

**Q: What is the blocking problem in 2PC?**
> Once a participant votes YES in Phase 1, it holds its locks and cannot release them until it receives the coordinator's Phase 2 decision. If the coordinator crashes after collecting YES votes but before sending COMMIT or ROLLBACK, all participants are stuck — holding locks indefinitely — until the coordinator recovers. This can freeze your entire system under failure.

**Q: Why can't a participant just time out and roll back if the coordinator doesn't respond?**
> Because the coordinator may have already sent COMMIT to other participants who have already committed. If one participant rolls back on timeout while others committed, you end up with partial commits — a worse state than the original inconsistency. Once a participant votes YES, it surrenders its ability to decide unilaterally.

**Q: Why do microservices avoid 2PC?**
> 2PC requires cross-service locks, creates a coordinator as a single point of failure, blocks all participants on coordinator failure, and creates tight coupling between independent services. All of these violate microservice principles of independence and availability. The Saga pattern with compensating transactions is the preferred alternative.

**Q: Does any production system use 2PC successfully?**
> Yes — Google Spanner uses 2PC internally but solves the coordinator failure problem by replicating the coordinator via Paxos and using TrueTime (GPS/atomic clocks) to bound uncertainty. CockroachDB uses 2PC with Raft-replicated coordinators. These systems accept 2PC's latency cost but engineer around its failure modes using consensus algorithms.

---

Say **next** for **Topic 9: Saga Pattern** — the microservices-native answer to distributed transactions, with choreography vs orchestration and compensating transactions 🚀
