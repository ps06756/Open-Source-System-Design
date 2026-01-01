# Design a Distributed Key-Value Store

Design a distributed key-value store like Redis or DynamoDB.

## 1. Requirements

### Functional Requirements
- `put(key, value)` - Store a value
- `get(key)` - Retrieve a value
- Optional: TTL support
- Optional: Compare-and-swap (CAS)

### Non-Functional Requirements
- High availability
- Scalability (handle petabytes)
- Tunable consistency
- Low latency (<10ms)

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│  Coordinator │────▶│      Nodes         │   │
│  └────────┘     │   (Router)   │     │  ┌────┐ ┌────┐    │   │
│                 └──────────────┘     │  │ N1 │ │ N2 │    │   │
│                                      │  └────┘ └────┘    │   │
│                                      │  ┌────┐ ┌────┐    │   │
│                                      │  │ N3 │ │ N4 │    │   │
│                                      │  └────┘ └────┘    │   │
│                                      └────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Key Components

### Data Partitioning (Consistent Hashing)

```
┌────────────────────────────────────────────────────────┐
│              Consistent Hashing Ring                   │
│                                                        │
│                      N1                                │
│                  ╱       ╲                             │
│               ╱             ╲                          │
│            N4                  N2                      │
│               ╲             ╱                          │
│                  ╲       ╱                             │
│                      N3                                │
│                                                        │
│  Key "user:123" → hash → position on ring → Node N2  │
│                                                        │
│  Virtual nodes: Each physical node has multiple       │
│  positions for better distribution                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Replication

```
Replication Factor (RF) = 3

Key "user:123" stored on:
- N2 (primary)
- N3 (replica)
- N4 (replica)

Walk clockwise on ring to find N replicas
```

### Quorum

```
N = 3 (replicas)
W = 2 (write quorum)
R = 2 (read quorum)

W + R > N guarantees strong consistency

Write: Must succeed on W nodes before returning
Read: Query R nodes, return latest version
```

## 4. Write Path

```
┌────────────────────────────────────────────────────────┐
│                    Write Path                          │
│                                                        │
│  Client ──▶ Coordinator ──┬──▶ Node 1 (write)        │
│                           ├──▶ Node 2 (write)        │
│                           └──▶ Node 3 (write)        │
│                                                        │
│  Wait for W=2 acknowledgments                         │
│  Return success to client                             │
│                                                        │
│  If node fails:                                        │
│  - Write to hinted handoff                            │
│  - Replay when node recovers                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Write-Ahead Log

```
1. Append to commit log (sequential write - fast)
2. Write to memtable (in-memory)
3. Acknowledge to client
4. Background: Flush memtable to SSTable (disk)
```

## 5. Read Path

```
┌────────────────────────────────────────────────────────┐
│                     Read Path                          │
│                                                        │
│  Client ──▶ Coordinator ──┬──▶ Node 1 (read)         │
│                           └──▶ Node 2 (read)         │
│                                                        │
│  Wait for R=2 responses                               │
│  Compare versions (vector clock)                      │
│  Return latest value                                  │
│  If conflict: return all versions for resolution     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Read from Storage

```
1. Check memtable (RAM)
2. Check bloom filter (quick "not here" check)
3. Check SSTables from newest to oldest
4. Return value with highest timestamp
```

## 6. Storage Engine

### LSM Tree (Log-Structured Merge Tree)

```
┌────────────────────────────────────────────────────────┐
│                    LSM Tree                            │
│                                                        │
│  Write Path:                                           │
│  ┌───────────┐                                        │
│  │ Memtable  │ ← Writes go here first                │
│  │  (RAM)    │                                        │
│  └─────┬─────┘                                        │
│        │ Flush when full                              │
│        ▼                                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐        │
│  │ SSTable   │  │ SSTable   │  │ SSTable   │        │
│  │  Level 0  │  │  Level 1  │  │  Level 2  │        │
│  └───────────┘  └───────────┘  └───────────┘        │
│                                                        │
│  Compaction merges SSTables, removes deleted         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Bloom Filter

```
Probabilistic "is key in SSTable?" check

- False positives possible (says yes, actually no)
- False negatives impossible (if says no, definitely no)
- Saves disk reads for missing keys
```

## 7. Conflict Resolution

### Vector Clocks

```
┌────────────────────────────────────────────────────────┐
│                  Vector Clocks                         │
│                                                        │
│  Track version per node:                              │
│  {A: 2, B: 1, C: 0}                                   │
│                                                        │
│  Node A writes: {A: 3, B: 1, C: 0}                   │
│  Node B writes: {A: 2, B: 2, C: 0}                   │
│                                                        │
│  Neither is ancestor of other = CONFLICT             │
│  Return both, let client resolve                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Last Write Wins (LWW)

```
Simple: Highest timestamp wins
Problem: Clock skew can lose writes
Used by: Cassandra (with caution)
```

## 8. Failure Handling

### Hinted Handoff

```
Node B is down:
1. Coordinator stores "hint" for B on another node
2. Hint contains: {key, value, target: B}
3. When B recovers, hints are replayed
```

### Anti-Entropy (Merkle Trees)

```
Periodic sync between replicas:
1. Build Merkle tree of data
2. Compare tree roots
3. If different, compare children
4. Sync only differing branches
```

## 9. Key Takeaways

1. **Consistent hashing** for partitioning
2. **Replication** for durability and availability
3. **Quorums** for tunable consistency
4. **LSM trees** for write-optimized storage
5. **Vector clocks** or LWW for conflict resolution
6. **Hinted handoff** for temporary failures

---

[Back to Beginner](../README.md) | Next: [Intermediate Designs](../../intermediate/01-twitter/README.md)
