# Data Partitioning

Data partitioning (sharding) is the technique of distributing data across multiple machines. It's essential for scaling beyond a single database.

## Table of Contents
- [Why Partition?](#why-partition)
- [Partitioning Strategies](#partitioning-strategies)
- [Consistent Hashing](#consistent-hashing)
- [Rebalancing](#rebalancing)
- [Partitioning Challenges](#partitioning-challenges)
- [Key Takeaways](#key-takeaways)

## Why Partition?

### Single Database Limits

```
┌─────────────────────────────────────────┐
│           Single Database               │
│                                         │
│  Data: 10 TB (disk limit: 2 TB)    ✗   │
│  Writes: 50K/sec (limit: 10K/sec)  ✗   │
│  Reads: 200K/sec (limit: 50K/sec)  ✗   │
│                                         │
└─────────────────────────────────────────┘
```

### With Partitioning

```
┌──────────────────────────────────────────────────────────┐
│                    Partitioned                           │
│                                                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐            │
│  │ Shard 1   │  │ Shard 2   │  │ Shard 3   │  ...       │
│  │ 2TB       │  │ 2TB       │  │ 2TB       │            │
│  │ 15K w/s   │  │ 15K w/s   │  │ 15K w/s   │            │
│  └───────────┘  └───────────┘  └───────────┘            │
│                                                          │
│  Total: 10TB+ capacity, 50K+ writes/sec                 │
└──────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Storage | More total disk capacity |
| Write throughput | Parallel writes to shards |
| Read throughput | Parallel reads from shards |
| Fault isolation | One shard failure doesn't affect others |

## Partitioning Strategies

### 1. Range Partitioning

Partition by key ranges.

```
┌─────────────────────────────────────────────────────────┐
│                    User Data                            │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Shard 1     │  │ Shard 2     │  │ Shard 3     │     │
│  │ A-H         │  │ I-P         │  │ Q-Z         │     │
│  │             │  │             │  │             │     │
│  │ alice       │  │ john        │  │ sarah       │     │
│  │ bob         │  │ kate        │  │ tom         │     │
│  │ charlie     │  │ mary        │  │ wendy       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Advantages:**
- Range queries are efficient (scan one shard)
- Easy to understand

**Disadvantages:**
- Hotspots (uneven distribution)
- Sequential keys go to same shard

**Use case:** Time-series data (partition by time range)

### 2. Hash Partitioning

Partition by hash of key.

```
┌─────────────────────────────────────────────────────────┐
│                    Hash Partitioning                    │
│                                                         │
│  user_id = "alice"                                      │
│      │                                                  │
│      ▼                                                  │
│  hash("alice") = 2847291                               │
│      │                                                  │
│      ▼                                                  │
│  2847291 % 3 = 1  ───▶  Shard 1                        │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Shard 0     │  │ Shard 1     │  │ Shard 2     │     │
│  │ hash%3=0    │  │ hash%3=1    │  │ hash%3=2    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

**Advantages:**
- Even distribution (no hotspots)
- Simple implementation

**Disadvantages:**
- Range queries must hit all shards
- Adding shards requires rehashing

**Use case:** User data, sessions, general key-value

### 3. Directory-Based Partitioning

Use a lookup service to map keys to shards.

```
┌─────────────────────────────────────────────────────────┐
│                Directory-Based                          │
│                                                         │
│  ┌─────────────────────────────────┐                   │
│  │        Lookup Service            │                   │
│  │  ┌─────────────────────────┐    │                   │
│  │  │ user_123 → Shard 2      │    │                   │
│  │  │ user_456 → Shard 1      │    │                   │
│  │  │ user_789 → Shard 3      │    │                   │
│  │  └─────────────────────────┘    │                   │
│  └─────────────────┬───────────────┘                   │
│                    │                                    │
│        ┌───────────┼───────────┐                       │
│        ▼           ▼           ▼                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                  │
│  │ Shard 1 │ │ Shard 2 │ │ Shard 3 │                  │
│  └─────────┘ └─────────┘ └─────────┘                  │
└─────────────────────────────────────────────────────────┘
```

**Advantages:**
- Flexible mapping (any key to any shard)
- Easy to rebalance

**Disadvantages:**
- Lookup service is SPOF
- Extra network hop

**Use case:** Complex mapping requirements

### 4. Composite Partitioning

Combine strategies for multi-level partitioning.

```
┌─────────────────────────────────────────────────────────┐
│                 Composite Partitioning                  │
│                                                         │
│  First level: Geographic (region)                       │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │   US-East    │  │   EU-West    │                    │
│  └──────┬───────┘  └──────┬───────┘                    │
│         │                  │                            │
│  Second level: Hash (user_id)                          │
│         │                  │                            │
│    ┌────┴────┐        ┌────┴────┐                      │
│    ▼         ▼        ▼         ▼                      │
│ ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                    │
│ │ S1  │  │ S2  │  │ S1  │  │ S2  │                    │
│ └─────┘  └─────┘  └─────┘  └─────┘                    │
└─────────────────────────────────────────────────────────┘
```

## Consistent Hashing

### The Problem with Simple Hashing

```
With 3 shards: hash(key) % 3

Add a 4th shard: hash(key) % 4

Almost all keys need to move! (expensive)
```

### How Consistent Hashing Works

```
Hash Ring:
                    0
                    │
         ┌──────────┴──────────┐
         │                     │
     N3 ●│                     │● N1
         │                     │
         │         ●           │
         │        K1           │
         │                     │
         │    ●                │
         │   K2                │
         │                     │
     N2 ●│                     │
         │                     │
         └─────────────────────┘

K1 → clockwise → N1
K2 → clockwise → N2

Add N4 between N2 and N3:
Only keys between N2 and N4 need to move!
```

### Virtual Nodes

Improve distribution with virtual nodes:

```
Without virtual nodes:
Ring: ─────N1────────────N2────────────N3─────

N1 handles 50% of keys (uneven!)

With virtual nodes (3 per node):
Ring: ─N1a──N2a──N3a──N1b──N2b──N3b──N1c──N2c──N3c─

Each node handles ~33% of keys (even distribution)
```

### Implementation Example

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes=None, virtual_nodes=100):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            self.ring[hash_val] = node
            self.sorted_keys.append(hash_val)
        self.sorted_keys.sort()

    def remove_node(self, node):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_val = self._hash(virtual_key)
            del self.ring[hash_val]
            self.sorted_keys.remove(hash_val)

    def get_node(self, key):
        if not self.ring:
            return None
        hash_val = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_val)
        if idx == len(self.sorted_keys):
            idx = 0
        return self.ring[self.sorted_keys[idx]]
```

## Rebalancing

When you add/remove shards, data needs to move.

### Strategies

**1. Fixed Partitions**
```
Create more partitions than nodes upfront:

1000 partitions, 10 nodes = 100 partitions per node

Add node 11: redistribute partitions
Each node gives up ~10 partitions
```

**2. Dynamic Partitioning**
```
Split partitions when too large:

┌─────────────────────┐
│     Partition 1     │  (10 GB - threshold reached)
│      (10 GB)        │
└─────────────────────┘
           │
           ▼ Split
┌──────────┐ ┌──────────┐
│  P1a     │ │  P1b     │
│  (5 GB)  │ │  (5 GB)  │
└──────────┘ └──────────┘
```

**3. Proportional Partitioning**
```
Partitions per node = total_size / target_size_per_partition

Node with more data → more partitions
```

### Rebalancing Considerations

| Consideration | Approach |
|---------------|----------|
| Minimize data movement | Consistent hashing |
| Maintain availability | Move one partition at a time |
| Throttle transfers | Limit bandwidth usage |
| Handle failures | Resume interrupted transfers |

## Partitioning Challenges

### 1. Hotspots

```
Problem: One shard gets disproportionate traffic

Example: Celebrity user_id always hashes to same shard

Solutions:
- Add salt: hash(user_id + random_suffix)
- Detect and split hot partitions
- Application-level load spreading
```

### 2. Cross-Shard Queries

```
Query: SELECT * FROM orders WHERE status = 'pending'

With user_id partitioning:
Must query ALL shards (scatter-gather)

┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard 1 │  │ Shard 2 │  │ Shard 3 │
│  query  │  │  query  │  │  query  │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┴────────────┘
                  │
                  ▼
            Aggregate results
```

### 3. Cross-Shard Transactions

```
Transfer $100 from user A (Shard 1) to user B (Shard 2)

Option 1: Two-Phase Commit (2PC)
- Coordinator asks both shards to prepare
- If both OK, commit; otherwise rollback
- Blocking, not partition-tolerant

Option 2: Saga Pattern
- Debit A, then credit B
- If credit fails, compensate (credit A back)
- Eventually consistent
```

### 4. Choosing the Right Partition Key

| Data Type | Good Partition Key | Bad Partition Key |
|-----------|-------------------|-------------------|
| User data | user_id | country (hotspots) |
| Orders | order_id or user_id | date (hotspots) |
| Logs | timestamp (range) | log_level (uneven) |
| Chat messages | chat_room_id | sender_id |

### 5. Secondary Indexes

```
Primary: user_id → shard

How to query by email?

Option 1: Local secondary index
Each shard indexes its own data
Query must hit all shards

Option 2: Global secondary index
Separate partitioned index
Write: update primary + index
Read: query index partition only
```

## Key Takeaways

1. **Partition for scale** - when single machine isn't enough
2. **Hash for distribution** - even spread across shards
3. **Range for locality** - efficient range queries
4. **Consistent hashing** - minimize rebalancing
5. **Choose partition key carefully** - affects query patterns
6. **Plan for cross-shard operations** - they're expensive

## Practice Questions

1. Design a sharding strategy for a social media posts table.
2. How would you handle a hot partition?
3. Explain why range partitioning can lead to hotspots.
4. How does adding a node affect data distribution with consistent hashing?

## Further Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter 6
- [Consistent Hashing Paper](https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf)
- [Amazon Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

---

Next: [Replication](../07-replication/README.md)
