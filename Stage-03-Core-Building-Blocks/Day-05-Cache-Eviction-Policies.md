# Cache Eviction Policies — Deep Dive

## Why This Deserves Its Own Session

We covered eviction policies briefly in Stage 3 Topic 4. But this topic comes up in interviews specifically — not just as "name the policies" but as "implement LRU", "compare LRU vs LFU for this workload", or "why does Redis approximate LRU instead of doing true LRU?" That level requires understanding the **data structures behind each policy**, not just the concept.

---

## The Core Problem

```
Cache has finite memory — say 1GB.
System generates more data than 1GB.

When cache is full and a new item must be inserted:
  Which existing item do you evict to make room?

Wrong choice:
  Evict a popular item → immediate cache miss for many users
  → DB gets hammered → latency spikes → bad UX

Right choice:
  Evict an item unlikely to be requested again
  → Cache miss rate stays low → DB stays calm → fast responses

Eviction policy = the algorithm for making this choice.
```

---

## Policy 1 — LRU (Least Recently Used)

### The Concept

```
Evict the item that was accessed least recently.

Intuition: if you haven't used it recently, you probably won't soon.
This works because most real access patterns have temporal locality
— recently accessed data tends to be accessed again soon.

Access sequence: A B C D A B E
Cache size: 3

State after each access:
  A     → [A]           cache: {A}
  B     → [B, A]        cache: {A, B}
  C     → [C, B, A]     cache: {A, B, C} — full
  D     → [D, C, B] A evicted  cache: {B, C, D}
  A     → [A, D, C] B evicted  cache: {A, C, D}
  B     → [B, A, D] C evicted  cache: {A, B, D}
  E     → [E, B, A] D evicted  cache: {A, B, E}
```

### The Data Structure — O(1) Implementation

Naive LRU using a sorted list is O(n) per operation — unusable at scale. The correct implementation uses **HashMap + Doubly Linked List**:

```
HashMap:    key → node pointer (O(1) lookup)
DLL:        ordered by recency (head=most recent, tail=least recent)

On GET(key):
  1. HashMap lookup → find node (O(1))
  2. Move node to head of DLL (O(1))
  3. Return value

On PUT(key, value):
  1. If key exists: update value, move node to head (O(1))
  2. If cache full: remove tail node + remove from HashMap (O(1))
  3. Create new node at head + add to HashMap (O(1))

Every operation is O(1) — no scanning required
```

```java
// LRU Cache — classic interview implementation
public class LRUCache<K, V> {

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final DoublyLinkedList<K, V> dll;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.dll = new DoublyLinkedList<>();
    }

    public V get(K key) {
        if (!map.containsKey(key)) return null;

        Node<K, V> node = map.get(key);
        dll.moveToHead(node);       // mark as most recently used
        return node.value;
    }

    public void put(K key, V value) {
        if (map.containsKey(key)) {
            Node<K, V> node = map.get(key);
            node.value = value;
            dll.moveToHead(node);   // update and mark as most recent
            return;
        }

        if (map.size() == capacity) {
            Node<K, V> lru = dll.removeTail();  // evict least recently used
            map.remove(lru.key);
        }

        Node<K, V> newNode = new Node<>(key, value);
        dll.addToHead(newNode);
        map.put(key, newNode);
    }

    // ── Internal Data Structures ───────────────────────────────

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private static class DoublyLinkedList<K, V> {
        Node<K, V> head; // most recently used
        Node<K, V> tail; // least recently used

        DoublyLinkedList() {
            // Sentinel nodes — simplify edge case handling
            head = new Node<>(null, null);
            tail = new Node<>(null, null);
            head.next = tail;
            tail.prev = head;
        }

        void addToHead(Node<K, V> node) {
            node.next = head.next;
            node.prev = head;
            head.next.prev = node;
            head.next = node;
        }

        void removeNode(Node<K, V> node) {
            node.prev.next = node.next;
            node.next.prev = node.prev;
        }

        void moveToHead(Node<K, V> node) {
            removeNode(node);
            addToHead(node);
        }

        Node<K, V> removeTail() {
            Node<K, V> lru = tail.prev;
            removeNode(lru);
            return lru;
        }
    }
}
```

**Memory layout:**
```
HashMap:  { "product:p-456" → Node1, "product:p-789" → Node2, "user:u-123" → Node3 }

DLL (most recent → least recent):
  [HEAD] ↔ [Node3: user:u-123] ↔ [Node1: product:p-456] ↔ [Node2: product:p-789] ↔ [TAIL]
                ↑                                                         ↑
           most recently                                           least recently
             accessed                                                accessed

GET "product:p-789":
  map lookup → Node2
  move Node2 to head

  [HEAD] ↔ [Node2: product:p-789] ↔ [Node3: user:u-123] ↔ [Node1: product:p-456] ↔ [TAIL]

Cache full, PUT new item:
  Remove tail → Node1 (product:p-456) evicted
  Remove from map
  Insert new node at head
```

### LRU Weakness — The Scan Problem

```
Scenario: cache holds 1000 popular products
          A one-time batch job reads 5000 rarely-accessed products

LRU evicts 1000 popular products to make room for
the 5000 batch items — items that will never be
accessed again after the batch.

After batch completes:
  Cache contains: 4000 batch items (useless)
  Lost: 1000 popular products (very useful)
  Cache hit rate crashes until repopulated

This is the cache pollution problem.
Solution: LFU, or separate cache for batch workloads.
```

---

## Policy 2 — LFU (Least Frequently Used)

### The Concept

```
Evict the item accessed the fewest times overall.

Intuition: popular items accumulate high counts and are protected.
Unpopular items stay at count=1 and are evicted first.

Access sequence: A A A B B C
Cache size: 2

Frequencies:
  A=3, B=2, C=1

Eviction choice: C (lowest frequency)

Works well for:
  Product catalog where top 100 products get 90% of traffic
  Static assets accessed by popularity
  Any access pattern where popularity is stable over time
```

### The Data Structure — O(1) LFU

Naive LFU is O(n) — scan all items to find minimum frequency. The O(1) implementation requires a clever structure:

```
Three components:
  1. HashMap<key, {value, frequency}>
     — lookup item and its current frequency

  2. HashMap<frequency, LinkedHashSet<key>>
     — all items at each frequency level (insertion-ordered set)

  3. minFreq integer
     — current minimum frequency (for O(1) eviction target)
```

```java
public class LFUCache<K, V> {

    private final int capacity;
    private int minFreq;
    private final Map<K, V> keyToValue;
    private final Map<K, Integer> keyToFreq;
    private final Map<Integer, LinkedHashSet<K>> freqToKeys;

    public LFUCache(int capacity) {
        this.capacity = capacity;
        this.minFreq = 0;
        this.keyToValue = new HashMap<>();
        this.keyToFreq  = new HashMap<>();
        this.freqToKeys = new HashMap<>();
    }

    public V get(K key) {
        if (!keyToValue.containsKey(key)) return null;
        incrementFrequency(key);    // access → increment count
        return keyToValue.get(key);
    }

    public void put(K key, V value) {
        if (capacity <= 0) return;

        if (keyToValue.containsKey(key)) {
            keyToValue.put(key, value);
            incrementFrequency(key);
            return;
        }

        if (keyToValue.size() >= capacity) {
            evictLFU();
        }

        keyToValue.put(key, value);
        keyToFreq.put(key, 1);
        freqToKeys.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFreq = 1;  // new item always starts at frequency 1
    }

    private void incrementFrequency(K key) {
        int freq = keyToFreq.get(key);

        // Remove from current frequency bucket
        freqToKeys.get(freq).remove(key);
        if (freqToKeys.get(freq).isEmpty()) {
            freqToKeys.remove(freq);
            if (minFreq == freq) minFreq++;  // update minFreq if needed
        }

        // Add to next frequency bucket
        int newFreq = freq + 1;
        keyToFreq.put(key, newFreq);
        freqToKeys.computeIfAbsent(newFreq, k -> new LinkedHashSet<>()).add(key);
    }

    private void evictLFU() {
        // O(1) — we know minFreq exactly
        LinkedHashSet<K> minFreqKeys = freqToKeys.get(minFreq);
        K evictKey = minFreqKeys.iterator().next(); // oldest among min-freq (FIFO tiebreak)

        minFreqKeys.remove(evictKey);
        if (minFreqKeys.isEmpty()) freqToKeys.remove(minFreq);

        keyToValue.remove(evictKey);
        keyToFreq.remove(evictKey);
    }
}
```

**Visual state:**
```
Cache size: 3
Access: A A A B B C D (D triggers eviction)

State before D:
  keyToValue: {A:valA, B:valB, C:valC}
  keyToFreq:  {A:3, B:2, C:1}
  freqToKeys: {1:[C], 2:[B], 3:[A]}
  minFreq:    1

PUT D (cache full):
  evictLFU() → minFreq=1 → evict C (frequency 1)
  freqToKeys: {2:[B], 3:[A]}
  minFreq = 1 (new item D starts at 1)

After PUT D:
  keyToValue: {A:valA, B:valB, D:valD}
  keyToFreq:  {A:3, B:2, D:1}
  freqToKeys: {1:[D], 2:[B], 3:[A]}
  minFreq:    1
```

### LFU Weakness — The Aging Problem

```
Product p-001 was popular 6 months ago:
  frequency count: 50,000

Product p-999 was just listed this week:
  frequency count: 500 (and growing fast)

LFU keeps p-001 forever because of historical count.
p-001 is no longer popular — frequency is stale.

Solution: Frequency Decay (Windowed LFU)
  Instead of total count, count accesses in last N hours/days
  Use sliding window or decay factor:
    freq = freq * decay_factor + recent_accesses
    decay_factor = 0.9 (10% decay per time period)
  
  Old popularity fades — new trending items get promoted
  Redis's LFU implementation uses this approach:
    lfu-decay-time: 1 (counter halved every 1 minute of inactivity)
    lfu-log-factor: 10 (logarithmic counter — prevents runaway growth)
```

---

## Policy 3 — MRU (Most Recently Used)

### The Concept

```
Evict the item accessed most recently.
Counter-intuitive — why evict what you JUST used?

When it makes sense:

Scenario: Sequential scan of a large dataset
  Cache size: 3
  Access pattern: 1, 2, 3, 4, 5, 6, 7... (never revisit)

With LRU:
  [1,2,3] → access 4 → evict 1 → [2,3,4]
  [2,3,4] → access 5 → evict 2 → [3,4,5]
  Keeps 2,3 which we'll never need again
  Hit rate: 0%

With MRU:
  [1,2,3] → access 4 → evict 3 (most recent!) → [1,2,4]
  [1,2,4] → access 5 → evict 4 → [1,2,5]
  Keeps 1,2 — still useless but different pattern
  
  For pure sequential scans, nothing helps much.
  But MRU prevents the LAST item from being immediately evicted
  when it might be the "current position" in the scan.

Stack-based access patterns:
  Function call cache — the most recently called function
  is the one currently executing — it will return soon.
  Once it returns (completed), evicting it is correct.

Real use cases:
  PostgreSQL buffer pool for sequential scans
  File system cache for backup/restore operations
  Rarely used in application-level caching
```

---

## Policy 4 — FIFO (First In, First Out)

### The Concept

```
Evict the item that was inserted first (oldest by insertion time).
Ignores access patterns entirely.

Queue structure: oldest → [item3] [item2] [item1] → newest

Access sequence: A B C D (cache size 3)
  Insert A → [A]
  Insert B → [B, A]
  Insert C → [C, B, A] — full
  Insert D → evict A (first inserted) → [D, C, B]

Simple but fundamentally flawed:
  A popular item inserted early = always evicted before a
  useless item inserted recently.
  
  Product p-001 inserted first: 1,000,000 accesses
  Product p-999 inserted last:  1 access
  Cache full → FIFO evicts p-001 → catastrophic miss

When FIFO is acceptable:
  All items have roughly equal access probability
  Simplicity matters more than optimal hit rate
  Items have a natural time-based expiry (TTL-like)
  Message processing queues (though those use deletion not eviction)

Implementation: simple queue — O(1) enqueue + dequeue
```

---

## Policy 5 — LIFO (Last In, First Out)

### The Concept

```
Evict the item inserted most recently.

Stack structure: newest items evicted first.

When it makes sense:
  Systems where older data is more reliable/valuable.
  
  Scenario: Streaming financial data
    t=0: confirmed price = $100.00 (verified)
    t=1: preliminary price = $100.05 (unconfirmed)
    t=2: preliminary price = $100.03 (unconfirmed)
    
    Cache full → evict newest (unconfirmed) data
    Keep oldest (confirmed) data — more trustworthy
  
  Undo/redo systems:
    Most recent uncommitted action should be undoable
    LIFO eviction aligns with undo semantics

In practice: almost never used in production caching.
Mostly theoretical — know it exists, understand when it fits.
```

---

## Policy 6 — Random Replacement

### The Concept

```
Evict a randomly chosen item.

No tracking overhead — no recency counts, no frequency counts.
No data structures beyond the cache itself.

Performance: surprisingly competitive with LRU for
             random access patterns where all items
             are equally likely to be accessed next.

Random access: LRU offers no advantage over random.
Temporal access (recent = likely reused): LRU wins.

Used in:
  CPU hardware TLB (Translation Lookaside Buffer)
    Hardware cache — random eviction avoids complex circuitry
  Some CDN edge caches — simplicity at scale
  A/B testing cache comparisons

Not used in:
  Application-level caching — LRU/LFU always better for real workloads
```

---

## Redis Eviction — Why It Approximates LRU/LFU

### The Problem with True LRU at Scale

```
True LRU requires:
  Doubly linked list: every node has prev + next pointers
  HashMap: key → node pointer

Redis stores millions of keys.
Per-key overhead of prev+next pointers = 16 bytes × 10M keys = 160MB

At Redis scale, this overhead is significant.
True LRU also requires a global lock when moving nodes —
even reads require a write to the DLL. Bad for concurrency.

Redis solution: probabilistic LRU approximation
```

### Redis LRU Approximation

```
Each Redis object stores a 24-bit LRU clock:
  lru_clock: seconds since last access (24-bit = wraps every 194 days)
  Overhead: 3 bytes per key vs 16 bytes for true LRU pointers

On eviction:
  Sample N random keys (default N=5, configurable via maxmemory-samples)
  From the sample, evict the one with the oldest lru_clock
  
  True LRU:          scan entire keyspace, find actual oldest
  Redis approx LRU:  sample 5, find oldest among 5

Why this works:
  With N=5 samples: good approximation of true LRU
  With N=10 samples: very close to true LRU
  With N=1 sample:   basically random replacement
  
  Error decreases as N increases — tune based on accuracy needs
  
  Config:
    maxmemory-samples 5   ← default
    maxmemory-samples 10  ← better accuracy, slightly more CPU
```

### Redis LFU Approximation

```
LFU requires tracking frequency counts.
Redis uses a probabilistic counter (Morris counter):

  8-bit counter (0-255) — not a true count
  
  Increment logic:
    If counter < 5:  always increment
    If counter < 10: 50% chance to increment
    If counter > 10: 1/(counter - 10) × factor chance to increment
    
  This gives logarithmic growth — frequent items get high counts
  but counter doesn't saturate quickly.
  
  Configured by: lfu-log-factor (default 10)
  
  Decay:
    Counter decays over time when key is not accessed
    Configured by: lfu-decay-time (default 1 minute)
    
    On each access: check time since last decrement
    Reduce counter by (elapsed_minutes / lfu-decay-time)
    
    This prevents old popular items from dominating forever.
```

---

## Comparing Policies — When Does Each Win?

```
Access Pattern          Best Policy   Why
──────────────────────────────────────────────────────────────
Temporal locality       LRU           Recently used = likely reused
Skewed popularity       LFU           Popular items accumulate count
Sequential scan         MRU or FIFO   LRU actively harmful (scan thrash)
All equal probability   Random/FIFO   No pattern to exploit
Time-ordered data       FIFO          Oldest = least valuable
New items more valuable LIFO          Newest = most relevant
Changing popularity     LFU + decay   Handles trending items
Mixed workload          2Q or ARC     Adaptive to pattern shifts
```

### ARC — Adaptive Replacement Cache

```
ARC (Adaptive Replacement Cache) — IBM Research
Automatically adapts between LRU and LFU behaviour.

Maintains 4 lists:
  T1: recently inserted, accessed once (LRU target)
  T2: accessed more than once (LFU target)
  B1: ghost entries for recently evicted from T1 (no data, just keys)
  B2: ghost entries for recently evicted from T2

Adaptation logic:
  Cache miss + key in B1 (was in T1, recently evicted):
    → Increase T2 size (more LFU bias)
    → Item was popular — shouldn't have evicted it so soon
    
  Cache miss + key in B2 (was in T2, recently evicted):
    → Increase T1 size (more LRU bias)
    → Item was accessed again — recency matters more

  Parameter p adjusts T1/T2 split dynamically

Result:
  ARC performs as well as the better of LRU/LFU for any workload
  Self-tuning — no manual configuration needed
  
Used by: ZFS filesystem cache, IBM DB2, some CDNs
Not in Redis (patent issues historically, now expired)
```

---

## Production Tuning — ShopSphere Redis Config

```bash
# Redis eviction configuration for ShopSphere

# Maximum memory
maxmemory 8gb

# Eviction policy
maxmemory-policy allkeys-lfu
# Why LFU for ShopSphere:
#   Product catalog: top 5% of products get 80% of traffic
#   LFU keeps popular products, evicts long-tail products
#   Better than LRU for skewed product access patterns

# LFU tuning
lfu-log-factor 10       # default — good balance
lfu-decay-time 1        # frequency halved every 1 minute of inactivity
                         # prevents old popular items dominating forever

# LRU sample size (used if switching to LRU policy)
maxmemory-samples 10    # more accurate approximation

# Lazy eviction — don't block on eviction
lazyfree-lazy-eviction yes  # eviction runs in background thread
lazyfree-lazy-expire yes    # TTL expiry in background
lazyfree-lazy-server-del yes
```

**Per-key TTL strategy alongside eviction:**
```java
// TTL-based expiry + eviction policy work together:

// Short TTL for volatile data — expires before eviction needed
redisTemplate.opsForValue()
    .set("search:results:" + queryHash, results, Duration.ofMinutes(2));

// Longer TTL for stable data — eviction policy handles if memory pressure
redisTemplate.opsForValue()
    .set("product:" + productId, product, Duration.ofMinutes(30));

// No TTL for persistent reference data — eviction policy only
redisTemplate.opsForValue()
    .set("category:" + categoryId, category);
// Eviction will remove this when memory is full

// volatile-lfu would only apply eviction to keys WITH TTL set
// allkeys-lfu applies eviction to ALL keys regardless of TTL
// ShopSphere uses allkeys-lfu to allow eviction of anything
```

---

## Cache Eviction in Other Systems

```
CPU L1/L2/L3 Cache:
  LRU pseudo-implementation (PLRU — simplified for hardware speed)
  Critical: microsecond decisions, no software overhead allowed

OS Page Cache (Linux):
  2Q algorithm — similar to ARC
  Pages split into "recently used" and "frequently used" queues
  Protects frequently accessed pages from scan thrash

PostgreSQL Buffer Pool:
  Clock-sweep algorithm (approximates LRU)
  Each buffer has a "usage count" (0-5)
  Clockhand sweeps — decrements count on pass
  Evicts first buffer with count=0
  Sequential scans get usage_count=1 (lower priority)
  Random page hits get usage_count incremented (higher priority)
  
Cassandra Row Cache:
  LRU
  Off-heap memory (JVM GC doesn't see it)
  
Nginx Proxy Cache:
  LRU with minimum_uses setting
  Protects from one-hit wonders polluting cache

Browser HTTP Cache:
  Not eviction-based — TTL/max-age based expiry
  When storage limit hit: LRU across all cached resources
```

---

## Interview Q&A

**Q: Implement an LRU cache with O(1) get and put.**
Use a HashMap for O(1) key lookup and a doubly linked list to maintain recency order. The HashMap maps keys to DLL nodes — lookup is O(1). On get, find the node via HashMap and move it to the head of the DLL — O(1). On put, if the key exists update it and move to head. If the cache is full, remove the tail node (LRU item) from both the DLL and HashMap, then insert the new node at the head. Java's LinkedHashMap provides this out of the box with `accessOrder=true` and override `removeEldestEntry`.

**Q: What is the difference between LRU and LFU and when would you choose each?**
LRU evicts the item accessed least recently — it bets that recently accessed items will be accessed again soon. It works well for workloads with temporal locality. LFU evicts the item accessed least frequently overall — it bets that rarely accessed items will remain rarely accessed. It works better for skewed workloads where a small fraction of items get the vast majority of traffic. The weakness of LRU is scan pollution — a one-time sequential scan evicts popular items. The weakness of LFU is aging — items popular in the past accumulate high counts and resist eviction even after becoming unpopular. For ShopSphere's product cache where top products are stably popular, LFU with frequency decay is the better choice.

**Q: Why does Redis approximate LRU instead of implementing true LRU?**
True LRU requires a doubly linked list where every node has prev and next pointers — 16 bytes per key overhead. At Redis scale with tens of millions of keys this becomes hundreds of megabytes of overhead for bookkeeping alone. Every read operation would also require a write to the linked list to update recency order, creating contention. Redis instead stores a 24-bit last-access timestamp per key — only 3 bytes — and on eviction samples a configurable number of random keys, evicting the one with the oldest timestamp. With the default sample size of 5 this closely approximates true LRU with minimal overhead. Increasing maxmemory-samples improves accuracy at the cost of slightly higher CPU usage during eviction.

**Q: What is cache scan pollution and how do you prevent it?**
Cache scan pollution occurs when a sequential access pattern — like a batch job reading millions of records once — evicts popular cached items to make room for items that will never be accessed again. With LRU, the popular items get evicted because they are now the oldest in the recency order after the scan fills the cache with new items. Prevention strategies include running batch workloads against a separate cache instance or bypassing the cache entirely for known sequential scans, using LFU instead of LRU so frequently accessed popular items are protected by their access count, or using ARC which automatically detects scan patterns and adjusts the eviction balance. In application code you can also explicitly set a very short TTL for batch-read items so they expire quickly without polluting the cache for long.

**Q: What are the eight Redis eviction policies and when would you use each?**
The eight policies split into three axes — scope, algorithm, and whether to target all keys or only keys with TTL. noeviction rejects new writes when full — only use if you must never lose data and prefer errors. allkeys-lru evicts the globally least recently used key — good general purpose for caches. volatile-lru applies LRU only to keys with TTL — keeps persistent reference data safe from eviction. allkeys-lfu evicts globally least frequently used — better than LRU for skewed access patterns like product catalogs. volatile-lfu applies LFU only to TTL keys. allkeys-random and volatile-random evict randomly — useful when access patterns are truly uniform. volatile-ttl evicts the key closest to expiry — useful when TTL values represent priority and you want the most-expired items gone first.

---

Say **"next"** to continue with **Stage 4 — Database Deep Dive**, or name any other topic you want to revisit.
