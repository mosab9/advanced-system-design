# Design a Distributed Cache (Memcached/Redis)

A fundamental building block for scalable systems.

---

## 1. Requirements

### Functional Requirements
- Put(key, value, TTL)
- Get(key) → value
- Delete(key)
- Support for expiration
- Optional: Increment/Decrement atomic operations

### Non-Functional Requirements
- Sub-millisecond latency
- High throughput (100K+ ops/sec per node)
- High availability
- Horizontal scalability
- Memory efficient

---

## 2. Core Components

### Single Node Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Cache Node                                  │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Network Layer                             │  │
│   │              (TCP/UDP, Protocol Parser)                     │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Hash Table                                │  │
│   │              (In-memory key-value store)                    │  │
│   │                                                              │  │
│   │   ┌─────────┬───────────────────────────────────────────┐   │  │
│   │   │  Hash   │  Bucket 0: key1→val1 → key7→val7 →...     │   │  │
│   │   │ Buckets │  Bucket 1: key2→val2                       │   │  │
│   │   │         │  Bucket 2: key3→val3 → key9→val9          │   │  │
│   │   │         │  ...                                       │   │  │
│   │   └─────────┴───────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                   Memory Allocator                          │  │
│   │                (Slab Allocator)                             │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                 Eviction Manager                            │  │
│   │              (LRU/LFU/TTL handling)                        │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Hash Table Design

### Structure
```
key → hash(key) → bucket_index → linked list/tree

Collision handling:
1. Chaining (linked list per bucket)
2. Open addressing (linear/quadratic probing)

For cache: Chaining is typically used
- Simpler deletion
- Predictable performance with high load
```

### Hash Function
```python
def hash_key(key):
    # MurmurHash3 or CityHash for speed
    # Should distribute evenly across buckets
    return murmur3_32(key) % num_buckets

# Resize when load factor > 0.75
# Double the buckets, rehash all keys
```

---

## 4. Memory Management (Slab Allocator)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Slab Allocator                                 │
│                                                                     │
│   Problem: Memory fragmentation with malloc/free                   │
│   Solution: Pre-allocate chunks of fixed sizes                     │
│                                                                     │
│   Slab Classes:                                                    │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │  Class 1:  64 bytes  [■][■][■][□][□]...                   │   │
│   │  Class 2:  128 bytes [■][■][□][□][□]...                   │   │
│   │  Class 3:  256 bytes [■][□][□][□][□]...                   │   │
│   │  Class 4:  512 bytes [■][■][■][□][□]...                   │   │
│   │  ...                                                       │   │
│   │  Class N:  1MB       [■][□][□]...                         │   │
│   └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ■ = Used slot                                                    │
│   □ = Free slot                                                    │
│                                                                     │
│   Allocation:                                                      │
│   1. Determine smallest slab class that fits                       │
│   2. Find free slot in that class                                 │
│   3. If no free slots, allocate new page or evict                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Eviction Policies

### LRU (Least Recently Used)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LRU Implementation                               │
│                                                                     │
│   Doubly Linked List + Hash Map                                    │
│                                                                     │
│   Hash Map: key → node pointer                                     │
│                                                                     │
│   Doubly Linked List (most recent → least recent):                 │
│   HEAD ←→ [K3] ←→ [K1] ←→ [K5] ←→ [K2] ←→ TAIL                    │
│            ↑                                                        │
│        Most recent                                                  │
│                                                                     │
│   GET K1:                                                          │
│   1. Find node via hash map O(1)                                   │
│   2. Move node to head O(1)                                        │
│   3. Return value                                                  │
│                                                                     │
│   PUT (when full):                                                 │
│   1. Remove tail node O(1)                                         │
│   2. Remove from hash map O(1)                                     │
│   3. Insert new node at head O(1)                                  │
│                                                                     │
│   All operations O(1)!                                             │
└─────────────────────────────────────────────────────────────────────┘
```

```python
class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key → node
        self.head = Node(0, 0)  # dummy head
        self.tail = Node(0, 0)  # dummy tail
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._move_to_head(node)
            return node.value
        return None

    def put(self, key, value):
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._move_to_head(node)
        else:
            if len(self.cache) >= self.capacity:
                # Evict LRU (tail)
                lru = self.tail.prev
                self._remove(lru)
                del self.cache[lru.key]

            node = Node(key, value)
            self._add_to_head(node)
            self.cache[key] = node
```

### LFU (Least Frequently Used)

```
Track frequency of access:
- More complex: O(log n) with heap, or O(1) with frequency buckets
- Better for skewed access patterns

Frequency Buckets:
freq=1: [K4, K7]
freq=2: [K1, K3]
freq=5: [K2]

Evict from lowest frequency bucket (K4 or K7)
```

### TTL-Based Expiration

```
Approaches:

1. Lazy Expiration:
   - Check TTL on access
   - Return null if expired, delete entry
   - Pros: Simple
   - Cons: Memory not freed until accessed

2. Active Expiration:
   - Background thread scans periodically
   - Delete expired entries
   - Pros: Frees memory proactively
   - Cons: CPU overhead

3. Hybrid (Redis approach):
   - Lazy on access
   - Active: Sample random keys periodically
   - If > 25% expired in sample, repeat
```

---

## 6. Distributed Cache Architecture

### Client-Side Sharding (Consistent Hashing)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Consistent Hashing                              │
│                                                                     │
│   Hash Ring (0 to 2^32):                                           │
│                                                                     │
│                        0                                            │
│                        │                                            │
│               Node A ──●────────────────── Node B                   │
│                       ╱                     ╲                       │
│                      ╱                       ╲                      │
│                     ╱         Key K1          ╲                     │
│                    ╱           ●───────────────╲                    │
│                   ●                             ●                   │
│               Node D                         Node C                 │
│                    ╲                           ╱                    │
│                     ╲                         ╱                     │
│                      ╲                       ╱                      │
│                       ╲                     ╱                       │
│                        ──────────●─────────                        │
│                               2^31                                  │
│                                                                     │
│   Key K1 → hash(K1) → position on ring → next node clockwise      │
│   K1 maps to Node B                                                │
│                                                                     │
│   Adding/Removing node: Only ~1/N keys need to move               │
└─────────────────────────────────────────────────────────────────────┘
```

### Virtual Nodes

```
Problem: Uneven distribution with few physical nodes

Solution: Each physical node has multiple virtual nodes

Physical Node A → Virtual: A1, A2, A3, A4, A5
Physical Node B → Virtual: B1, B2, B3, B4, B5

Ring now has 10 positions (more even distribution)

┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   With virtual nodes:                                              │
│                                                                     │
│       A1        B2                                                 │
│         ●──────●                                                   │
│        ╱        ╲                                                  │
│    B1 ●          ● A2                                              │
│      │            │                                                │
│    A3 ●          ● B3                                              │
│        ╲        ╱                                                  │
│         ●──────●                                                   │
│       B4        A4                                                 │
│                                                                     │
│   More even key distribution                                       │
│   Typical: 100-200 virtual nodes per physical node                │
└─────────────────────────────────────────────────────────────────────┘
```

### Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │                         Clients                              │ │
│   │   ┌──────────────────────────────────────────────────────┐  │ │
│   │   │  Consistent Hash Ring (client library)              │  │ │
│   │   └──────────────────────────────────────────────────────┘  │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                            │                                        │
│         ┌──────────────────┼──────────────────┐                    │
│         │                  │                  │                    │
│         ▼                  ▼                  ▼                    │
│   ┌───────────┐     ┌───────────┐      ┌───────────┐              │
│   │  Node 1   │     │  Node 2   │      │  Node 3   │              │
│   │           │     │           │      │           │              │
│   │ Keys: 0-33│     │Keys: 34-66│      │Keys: 67-99│              │
│   └───────────┘     └───────────┘      └───────────┘              │
│                                                                     │
│   Each node is independent (shared-nothing)                        │
│   Client determines which node to contact                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. High Availability

### Replication

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Replication Strategy                              │
│                                                                     │
│   Option 1: Master-Replica (Redis)                                 │
│                                                                     │
│   ┌─────────────┐     Async replication    ┌─────────────┐        │
│   │   Master    │────────────────────────▶│   Replica   │        │
│   │  (Writes)   │                          │  (Reads)    │        │
│   └─────────────┘                          └─────────────┘        │
│                                                                     │
│   Option 2: Multi-Master                                           │
│                                                                     │
│   ┌─────────────┐           ┌─────────────┐                       │
│   │   Node 1    │◀─────────▶│   Node 2    │                       │
│   │  (Primary)  │  Sync     │  (Primary)  │                       │
│   └─────────────┘           └─────────────┘                       │
│                                                                     │
│   Trade-off:                                                       │
│   - Async: Better performance, potential data loss                │
│   - Sync: Stronger consistency, higher latency                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Failure Detection & Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Redis Sentinel                                   │
│                                                                     │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│   │ Sentinel 1  │  │ Sentinel 2  │  │ Sentinel 3  │               │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │
│          │                │                │                       │
│          └────────────────┼────────────────┘                       │
│                           │                                         │
│          Heartbeat/Monitor│                                         │
│                           ▼                                         │
│   ┌─────────────┐           ┌─────────────┐                       │
│   │   Master    │──────────▶│   Replica   │                       │
│   └─────────────┘ Replication└─────────────┘                       │
│                                                                     │
│   If Master fails:                                                 │
│   1. Sentinels detect via missing heartbeats                       │
│   2. Quorum agrees master is down                                  │
│   3. Promote replica to master                                     │
│   4. Redirect clients to new master                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Cache Patterns

### Cache-Aside

```
Read:
┌────────┐    1. Get    ┌─────────┐
│ Client │─────────────▶│  Cache  │ Miss
└────────┘              └─────────┘
    │                        │
    │ 2. Get from DB         │
    ▼                        │
┌────────┐                   │
│   DB   │                   │
└────────┘                   │
    │                        │
    │ 3. Store in cache      │
    └────────────────────────┘
```

### Write-Through

```
Write:
┌────────┐    1. Write   ┌─────────┐
│ Client │──────────────▶│  Cache  │
└────────┘               └─────────┘
                              │
                    2. Write to DB
                              │
                              ▼
                         ┌────────┐
                         │   DB   │
                         └────────┘

Data always consistent between cache and DB
Higher write latency
```

### Write-Behind (Write-Back)

```
Write:
┌────────┐    1. Write   ┌─────────┐
│ Client │──────────────▶│  Cache  │
└────────┘               └─────────┘
    │                         │
    │ Return immediately      │ 2. Async batch write
    │                         ▼
                         ┌────────┐
                         │   DB   │
                         └────────┘

Lower write latency
Risk of data loss if cache fails before DB write
```

---

## 9. Cache Stampede Prevention

```
Problem: Cache miss → all requests hit DB simultaneously

┌─────────────────────────────────────────────────────────────────────┐
│                    Cache Stampede                                   │
│                                                                     │
│   Key expires at 12:00:00                                          │
│                                                                     │
│   12:00:01: Request 1 → Cache miss → Query DB                      │
│   12:00:01: Request 2 → Cache miss → Query DB                      │
│   12:00:01: Request 3 → Cache miss → Query DB                      │
│   ...                                                               │
│   12:00:01: Request 1000 → Cache miss → Query DB                   │
│                                                                     │
│   DB overwhelmed!                                                  │
│                                                                     │
│   Solutions:                                                       │
│                                                                     │
│   1. Locking: Only one request fetches, others wait                │
│   2. Probabilistic Early Expiration                                │
│   3. Background Refresh                                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Locking Solution

```python
def get_with_lock(key):
    value = cache.get(key)
    if value is not None:
        return value

    # Try to acquire lock
    lock_key = f"lock:{key}"
    if cache.set(lock_key, "1", nx=True, ex=10):  # Got lock
        try:
            value = db.get(key)
            cache.set(key, value, ex=300)
            return value
        finally:
            cache.delete(lock_key)
    else:
        # Another request is fetching, wait and retry
        time.sleep(0.1)
        return get_with_lock(key)  # Retry
```

### Probabilistic Early Refresh

```python
def get_with_early_refresh(key):
    value, ttl = cache.get_with_ttl(key)

    if value is None:
        # Cache miss, fetch from DB
        return fetch_and_cache(key)

    # Probabilistically refresh before expiration
    # Earlier refresh = higher probability
    if should_refresh(ttl):
        # Refresh in background
        async_refresh(key)

    return value

def should_refresh(ttl):
    # When TTL < 60s, increasing probability
    if ttl > 60:
        return False
    probability = 1 - (ttl / 60)
    return random.random() < probability
```

---

## 10. Trade-offs

| Decision | Options | Trade-off |
|----------|---------|-----------|
| Eviction | LRU vs LFU | Recency vs Frequency |
| Replication | Sync vs Async | Consistency vs Latency |
| Sharding | Client vs Proxy | Simplicity vs Flexibility |
| Expiration | Lazy vs Active | Memory vs CPU |
| Consistency | Strong vs Eventual | Availability vs Correctness |

---

## 11. Interview Tips

**Key Points:**
1. Hash table implementation (O(1) operations)
2. Memory management (slab allocator)
3. LRU implementation (doubly linked list + hash map)
4. Consistent hashing for distribution
5. Cache patterns (cache-aside, write-through)
6. Stampede prevention

**Questions to Ask:**
- What's the expected hit rate?
- Read/write ratio?
- Single region or global?
- Consistency requirements?
