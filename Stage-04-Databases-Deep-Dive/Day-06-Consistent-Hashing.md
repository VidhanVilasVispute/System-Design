# Stage 4 — Databases Deep Dive
## Topic 6: Consistent Hashing — Node Assignment, Minimal Reshuffling, Virtual Nodes

---

## 1. The Problem With Naive Hashing

From the last topic, hash-based sharding:

```
shard_index = hash(user_id) % num_shards
```

This works perfectly until you need to add or remove a shard.

**You have 4 shards. You add a 5th.**

```
Before: hash(key) % 4
After:  hash(key) % 5
```

Almost every key remaps to a different shard. If you have 100M keys, approximately 80M of them move. That's a catastrophic data migration — weeks of backfill, dual-write complexity, service degradation.

**You have 4 shards. One dies.**

```
Before: hash(key) % 4
After:  hash(key) % 3
```

Same problem — 75% of keys remap. The death of one node causes a global data shuffle.

Consistent hashing solves this: **when a node is added or removed, only K/N keys move**, where K is the number of keys and N is the number of nodes. With 100M keys and 4 nodes, adding a 5th node moves only ~20M keys — exactly the ones that belong to the new node.

---

## 2. The Hash Ring

Consistent hashing maps both **nodes and keys** onto the same circular space — a ring of values from 0 to 2³² - 1 (or 0 to 2¹²⁸ for MD5/SHA).

### Building the Ring

1. Hash each node's identifier to a position on the ring

```
hash("shard-0") = 12
hash("shard-1") = 45
hash("shard-2") = 78
hash("shard-3") = 95

Ring (0 → 100, simplified):
  0────12────45────78────95────(back to 0)
       S0    S1    S2    S3
```

2. To find which node owns a key, hash the key and **walk clockwise** until you hit a node

```
hash("user:42")   = 30  → walk clockwise → hits S1 at 45  → S1 owns it
hash("user:99")   = 60  → walk clockwise → hits S2 at 78  → S2 owns it
hash("user:5000") = 97  → walk clockwise → wraps around  → hits S0 at 12 → S0 owns it
```

Each node owns all keys from its position back to the previous node (counter-clockwise). Node S1 owns keys in range (12, 45].

---

## 3. Adding a Node — Minimal Reshuffling

You add Shard 4, which hashes to position 60:

```
Before:  0────12────45────78────95────
               S0    S1    S2    S3

After:   0────12────45────60────78────95────
               S0    S1    S4    S2    S3
```

S4 lands between S1 (45) and S2 (78). S4 now owns keys in range (45, 60].

**Only the keys in (45, 60] move** — those keys were owned by S2 before. S2 gives up that slice to S4.

Everything else stays exactly where it was. S0, S1, S3 are completely unaffected.

---

## 4. Removing a Node — Minimal Reshuffling

S1 (position 45) dies:

```
Before:  0────12────45────60────78────95────
               S0    S1    S4    S2    S3

After:   0────12────60────78────95────
               S0    S4    S2    S3
```

S1's range (12, 45] now has no owner — the next node clockwise, S4, absorbs it.

**Only S1's keys move** — they go to S4. Everything else is untouched.

---

## 5. The Problem With Basic Consistent Hashing — Non-Uniform Distribution

With only 4 nodes, the hash function might place them unevenly:

```
Ring: 0────5────50────51────99────
           S0   S1    S2    S3
```

S1 owns (5, 50] — 45% of the ring.  
S2 owns (50, 51] — 1% of the ring.  
S3 owns (51, 99] — 48% of the ring.  
S0 owns (99, 5] — 6% of the ring (wrapping around).

S1 and S3 are overloaded. S2 is nearly idle. This is the load imbalance problem with naive consistent hashing.

---

## 6. Virtual Nodes — The Real Solution

Instead of placing each physical node once on the ring, place it **V times** (V = number of virtual nodes per physical node, typically 100–200).

```java
for (int i = 0; i < VIRTUAL_NODES; i++) {
    int position = hash("shard-0-vnode-" + i);
    ring.put(position, physicalNode0);
}
```

Each physical node now has 150 positions scattered around the ring. The ring looks like:

```
0──S2──S0──S3──S1──S2──S0──S1──S3──S0──S2──S3──S1── ... ──100
```

Physical nodes interleave across the ring. By the law of large numbers, each node ends up owning roughly equal portions of the ring regardless of where its actual hash values fall.

### Benefits of Virtual Nodes

**Even load distribution:** 150 virtual nodes per physical node → statistical evenness.

**Proportional scaling:** A more powerful machine gets more virtual nodes → naturally handles more keys.

**Smooth node addition:** When a new node joins with 150 virtual nodes, it takes small slices from 150 different existing nodes — load is redistributed gradually rather than one big chunk from one neighbor.

**Better failure handling:** When a node dies, its 150 virtual node positions are absorbed by 150 different neighbors — no single node absorbs all the failed node's traffic.

---

## 7. Data Replication on the Ring

In distributed systems (Cassandra, DynamoDB), you don't just want one copy of data. You want N replicas.

**Replication strategy:** For each key, the **first N nodes clockwise** from the key's position are its replicas.

```
Ring: ─── S0(12) ─── S1(45) ─── S2(78) ─── S3(95) ───

Key hashes to position 30. Replication factor = 3.
Walk clockwise from 30:
  1st node: S1 (45)  → primary replica
  2nd node: S2 (78)  → replica 2
  3rd node: S3 (95)  → replica 3
```

The key lives on S1, S2, S3. S0 holds no copy. If S1 dies, S2 is still available and takes over as primary.

This is exactly how **Apache Cassandra** implements replication — `SimpleStrategy` with replication factor 3 uses this ring-walking approach.

---

## 8. Read/Write Quorum on the Ring

With 3 replicas (N=3), you can tune consistency:

| Setting | Writes required | Reads required | Guarantee |
|---|---|---|---|
| ONE | 1 node ACK | 1 node | Fastest, weakest consistency |
| QUORUM | 2 node ACKs | 2 nodes | Strong consistency (W+R > N) |
| ALL | 3 node ACKs | 3 nodes | Strongest, least available |

**W + R > N guarantees strong consistency** — the read set and write set always overlap, so you always read at least one node that has the latest write.

`QUORUM` write + `QUORUM` read: 2 + 2 > 3 ✅ — at least one node in the read set saw the write.

---

## 9. Where Consistent Hashing is Used in Practice

| System | Usage |
|---|---|
| **Apache Cassandra** | Data distribution across nodes, replication |
| **Amazon DynamoDB** | Internal partition assignment |
| **Amazon S3** | Object → storage node mapping |
| **Memcached / Redis Cluster** | Key → cache node routing |
| **Nginx / HAProxy** | Consistent hash load balancing (sticky sessions without cookies) |
| **Chord DHT** | Peer-to-peer key lookup protocol |
| **Envoy Proxy** | Ring hash load balancing policy |

---

## 10. ShopSphere Mapping

### Redis Cluster for Session/Cart Caching

ShopSphere uses Redis for cart data and session tokens. Redis Cluster uses consistent hashing with **16384 hash slots** (not virtual nodes — a fixed-slot variant):

```
hash_slot = CRC16(key) % 16384

Slot ranges assigned to nodes:
Node 0: slots 0–5460
Node 1: slots 5461–10922
Node 2: slots 10923–16383
```

Adding a Redis node → reassign some slot ranges. Only keys in those slots move.

### Cart Service — Consistent Hash Load Balancing

When a user's cart request arrives at the Cart Service, route to the same upstream instance for that user (to hit warm local cache):

```yaml
# Envoy / Spring Cloud LoadBalancer config
loadBalancer:
  type: CONSISTENT_HASH
  hashHeader: X-User-Id
```

Same `user_id` always routes to the same Cart Service instance → L1 in-memory cache hit. If that instance goes down, only that user's requests reroute (and cold-start on another instance).

### Kafka Partitioning

Kafka uses consistent-hashing-style partitioning:

```java
// Kafka producer routes by key
producer.send(new ProducerRecord<>("order-events", userId.toString(), orderEvent));
// hash(userId) % numPartitions → always same partition for same user
// preserves ordering of events per user
```

---

## 11. Interview Q&A

**Q: Why does consistent hashing minimize data movement when nodes change?**
A: In naive modular hashing (`hash(key) % N`), changing N remaps nearly all keys because the modulus changes globally. In consistent hashing, both nodes and keys are on the same ring. A new node only takes ownership of the range between itself and the previous node — only keys in that range move. Mathematically, adding one node to an N-node ring moves only 1/(N+1) of total keys, regardless of ring size.

**Q: What problem do virtual nodes solve?**
A: Basic consistent hashing with few nodes causes uneven load distribution — hash values cluster unevenly on the ring, making some nodes own much more of the keyspace than others. Virtual nodes assign each physical node V positions on the ring. With enough virtual nodes (~150), statistical distribution ensures each physical node owns roughly 1/N of the keyspace. They also allow heterogeneous clusters — a more powerful node gets more virtual nodes and proportionally more data.

**Q: How does Cassandra use consistent hashing?**
A: Cassandra places each node's token (position) on a ring. A row's partition key is hashed to a position, and the row is stored on the first N nodes clockwise from that position (where N is the replication factor). With virtual nodes enabled (vnodes), each physical node has 256 token positions scattered around the ring. This provides automatic load balancing and makes adding/removing nodes seamless — the node's 256 positions each absorb from or release to different neighbors.

**Q: What is the difference between W + R > N and eventual consistency?**
A: W + R > N is a quorum condition that guarantees strong consistency — the read set and write set always share at least one node, so a read will always see the latest committed write. Eventual consistency relaxes this — writes go to one or more nodes, reads come from potentially different nodes, and the system converges to consistency over time. Eventual consistency is faster and more available; quorum reads are slower but never return stale data.

**Q: Can consistent hashing prevent hot spots entirely?**
A: No. Consistent hashing distributes keys evenly across nodes, but if one specific key is accessed far more than others (celebrity problem — one viral product), all traffic for that key still hits the same node(s). Consistent hashing solves the *distribution* problem but not the *access frequency* problem. Hot keys require application-layer solutions: caching (Redis), read replicas for that specific shard, or breaking the hot key into sub-keys with aggregation.

---

Ready for **Topic 7: NoSQL Categories — Key-Value, Document, Column-Family, Graph, and when to use each**?
