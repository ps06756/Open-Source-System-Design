# Design a Distributed Cache

Design a distributed caching system like Redis or Memcached.

## 1. Requirements

### Functional Requirements
- Get/Set/Delete operations
- TTL (Time To Live) support
- Atomic operations (INCR, DECR)
- Data structures (strings, lists, sets, hashes)
- Pub/Sub messaging

### Non-Functional Requirements
- Sub-millisecond latency
- High throughput (millions ops/sec)
- High availability
- Linear scalability
- Scale: 100TB+ data, millions QPS

### Capacity Estimation

```
Data: 100 TB across cluster
Operations: 10M ops/sec
Average key: 50 bytes
Average value: 1 KB
Memory per node: 128 GB
Nodes needed: 100 TB / 128 GB ≈ 800 nodes
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐                                                   │
│  │ Client │                                                   │
│  └───┬────┘                                                   │
│      │                                                         │
│      ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    Client Library                        │  │
│  │  (Consistent hashing, connection pooling, routing)      │  │
│  └────────────────────────┬────────────────────────────────┘  │
│                           │                                    │
│       ┌───────────────────┼───────────────────┐               │
│       │                   │                   │               │
│       ▼                   ▼                   ▼               │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐           │
│  │ Node 1  │        │ Node 2  │        │ Node N  │           │
│  │ Master  │        │ Master  │        │ Master  │           │
│  └────┬────┘        └────┬────┘        └────┬────┘           │
│       │                  │                  │                 │
│       ▼                  ▼                  ▼                 │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐           │
│  │Replica 1│        │Replica 2│        │Replica N│           │
│  └─────────┘        └─────────┘        └─────────┘           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Data Partitioning

```
┌────────────────────────────────────────────────────────────────┐
│                Consistent Hashing                              │
│                                                                │
│  Hash ring with virtual nodes:                                │
│                                                                │
│                   0                                            │
│               ╱       ╲                                        │
│           N3(v2)      N1(v1)                                  │
│          ╱               ╲                                     │
│       N2(v1)              N1(v2)                              │
│          ╲               ╱                                     │
│           N3(v1)      N2(v2)                                  │
│               ╲       ╱                                        │
│                  2^32                                          │
│                                                                │
│  Virtual nodes for even distribution:                         │
│  - Each physical node has 100-200 virtual nodes              │
│  - Key hashes to position, clockwise to next node           │
│                                                                │
│  Adding/removing nodes:                                       │
│  - Only affects neighboring nodes' data                      │
│  - ~1/N data moves when N nodes present                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Consistent Hashing Implementation

```python
import hashlib
from sortedcontainers import SortedDict

class ConsistentHash:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = SortedDict()
        self.nodes = set()

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        """Hash key to position on ring"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        """Add node with virtual nodes"""
        self.nodes.add(node)

        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            position = self._hash(virtual_key)
            self.ring[position] = node

    def remove_node(self, node):
        """Remove node and its virtual nodes"""
        self.nodes.discard(node)

        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            position = self._hash(virtual_key)
            del self.ring[position]

    def get_node(self, key):
        """Get node responsible for key"""
        if not self.ring:
            return None

        position = self._hash(key)

        # Find first node clockwise from position
        idx = self.ring.bisect_right(position)
        if idx >= len(self.ring):
            idx = 0

        return self.ring.values()[idx]
```

## 4. Memory Management

```
┌────────────────────────────────────────────────────────────────┐
│                  Memory Management                             │
│                                                                │
│  Storage structures:                                           │
│  1. Hash table: key → pointer to value                        │
│  2. Value storage: actual data with metadata                  │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Entry structure:                                        │  │
│  │  ┌────────┬────────┬────────┬────────┬────────────────┐│  │
│  │  │ Key    │ Value  │  TTL   │ Flags  │ Next pointer   ││  │
│  │  │ (hash) │ (ptr)  │ (unix) │        │ (for chaining) ││  │
│  │  └────────┴────────┴────────┴────────┴────────────────┘│  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Eviction policies:                                            │
│  - LRU (Least Recently Used)                                 │
│  - LFU (Least Frequently Used)                               │
│  - Random                                                      │
│  - TTL-based (expire oldest)                                 │
│                                                                │
│  Memory optimizations:                                         │
│  - Slab allocation (fixed-size chunks)                       │
│  - Memory pooling                                             │
│  - Compression for large values                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### LRU Implementation

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key):
        if key not in self.cache:
            return None

        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]['value']

    def put(self, key, value, ttl=None):
        if key in self.cache:
            self.cache.move_to_end(key)
        else:
            if len(self.cache) >= self.capacity:
                # Evict least recently used
                self.cache.popitem(last=False)

        expires_at = time.time() + ttl if ttl else None
        self.cache[key] = {
            'value': value,
            'expires_at': expires_at
        }

    def delete(self, key):
        if key in self.cache:
            del self.cache[key]
            return True
        return False
```

## 5. Replication

```
┌────────────────────────────────────────────────────────────────┐
│                   Replication                                  │
│                                                                │
│  Master-Replica model:                                        │
│  - Writes go to master                                        │
│  - Async replication to replicas                             │
│  - Reads can be served by replicas                           │
│                                                                │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│  │  Master  │────▶│ Replica1 │     │ Replica2 │              │
│  └────┬─────┘     └──────────┘     └──────────┘              │
│       │                  ▲               ▲                    │
│       └──────────────────┴───────────────┘                    │
│            Async replication                                   │
│                                                                │
│  Replication log:                                              │
│  1. Master writes to in-memory + append to log               │
│  2. Replica pulls log entries                                 │
│  3. Apply entries to local state                             │
│                                                                │
│  Failover:                                                     │
│  1. Replica detects master failure (heartbeat timeout)       │
│  2. Sentinel or cluster promotes replica to master           │
│  3. Other replicas reconnect to new master                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. TTL Management

```python
class TTLManager:
    def __init__(self):
        self.expiry_buckets = {}  # second -> set of keys
        self.key_expiry = {}       # key -> expiry_time

    def set_ttl(self, key, ttl_seconds):
        """Set expiry for key"""
        expiry_time = int(time.time()) + ttl_seconds

        # Remove old expiry if exists
        if key in self.key_expiry:
            old_time = self.key_expiry[key]
            self.expiry_buckets.get(old_time, set()).discard(key)

        # Set new expiry
        self.key_expiry[key] = expiry_time
        if expiry_time not in self.expiry_buckets:
            self.expiry_buckets[expiry_time] = set()
        self.expiry_buckets[expiry_time].add(key)

    def expire_keys(self, cache):
        """Called periodically to expire keys"""
        current_time = int(time.time())

        # Get all buckets up to current time
        expired_buckets = [t for t in self.expiry_buckets if t <= current_time]

        for bucket_time in expired_buckets:
            keys = self.expiry_buckets.pop(bucket_time, set())
            for key in keys:
                if self.key_expiry.get(key) == bucket_time:
                    cache.delete(key)
                    del self.key_expiry[key]

    def is_expired(self, key):
        """Check if key is expired"""
        if key not in self.key_expiry:
            return False
        return time.time() >= self.key_expiry[key]
```

## 7. Cache Operations

```python
class CacheNode:
    def __init__(self, config):
        self.data = {}
        self.ttl_manager = TTLManager()
        self.lock = threading.RLock()

    def get(self, key):
        """Get value for key"""
        with self.lock:
            if key not in self.data:
                return None

            if self.ttl_manager.is_expired(key):
                self.delete(key)
                return None

            return self.data[key]

    def set(self, key, value, ttl=None):
        """Set key-value pair"""
        with self.lock:
            self.data[key] = value
            if ttl:
                self.ttl_manager.set_ttl(key, ttl)
            return True

    def delete(self, key):
        """Delete key"""
        with self.lock:
            if key in self.data:
                del self.data[key]
                return True
            return False

    def incr(self, key, delta=1):
        """Atomic increment"""
        with self.lock:
            if key not in self.data:
                self.data[key] = 0
            self.data[key] += delta
            return self.data[key]

    def mget(self, keys):
        """Get multiple keys"""
        results = {}
        with self.lock:
            for key in keys:
                results[key] = self.get(key)
        return results
```

## 8. Client Library

```python
class CacheClient:
    def __init__(self, nodes):
        self.consistent_hash = ConsistentHash(nodes)
        self.connections = {}  # node -> connection pool

    def _get_connection(self, node):
        """Get connection from pool"""
        if node not in self.connections:
            self.connections[node] = ConnectionPool(node)
        return self.connections[node].get()

    def get(self, key):
        """Get value from cache"""
        node = self.consistent_hash.get_node(key)
        conn = self._get_connection(node)
        try:
            return conn.get(key)
        finally:
            self.connections[node].release(conn)

    def set(self, key, value, ttl=None):
        """Set value in cache"""
        node = self.consistent_hash.get_node(key)
        conn = self._get_connection(node)
        try:
            return conn.set(key, value, ttl)
        finally:
            self.connections[node].release(conn)

    def mget(self, keys):
        """Get multiple keys (may be on different nodes)"""
        # Group keys by node
        node_keys = {}
        for key in keys:
            node = self.consistent_hash.get_node(key)
            if node not in node_keys:
                node_keys[node] = []
            node_keys[node].append(key)

        # Fetch in parallel from each node
        results = {}
        with ThreadPoolExecutor() as executor:
            futures = {}
            for node, keys in node_keys.items():
                future = executor.submit(self._mget_from_node, node, keys)
                futures[future] = node

            for future in as_completed(futures):
                results.update(future.result())

        return results
```

## 9. Key Takeaways

1. **Consistent hashing** - even distribution, minimal reshuffling
2. **Virtual nodes** - handle uneven load and node differences
3. **LRU eviction** - manage memory when full
4. **Async replication** - high availability without latency hit
5. **TTL management** - bucket-based expiration
6. **Connection pooling** - reduce connection overhead

---

Next: [Distributed Queue](../06-distributed-queue/README.md)
