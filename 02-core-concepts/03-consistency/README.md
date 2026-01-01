# Consistency Models

Consistency defines the guarantees about when and how updates become visible to readers. Understanding consistency models is crucial for designing distributed systems.

## Table of Contents
- [What is Consistency?](#what-is-consistency)
- [Strong Consistency](#strong-consistency)
- [Eventual Consistency](#eventual-consistency)
- [Other Consistency Models](#other-consistency-models)
- [Consistency in Practice](#consistency-in-practice)
- [Key Takeaways](#key-takeaways)

## What is Consistency?

In distributed systems, consistency refers to whether all nodes see the same data at the same time.

```
Without Consistency Guarantees:
┌────────┐     Write X=1     ┌────────┐
│ Client │─────────────────▶│ Node A │
└────────┘                   └────────┘
                                  │
                                  │ Replication (delay)
                                  ▼
┌────────┐     Read X        ┌────────┐
│ Client │─────────────────▶│ Node B │  Returns X=0 (stale!)
└────────┘                   └────────┘
```

### The Spectrum

```
Strong                                               Weak
Consistency                                     Consistency
    │                                                 │
    ▼                                                 ▼
┌───────┬──────────┬───────────┬──────────┬─────────┐
│Linear-│Sequential│  Causal   │  Read    │Eventual │
│izable │          │           │  Your    │         │
│       │          │           │  Writes  │         │
└───────┴──────────┴───────────┴──────────┴─────────┘
    │                                                 │
    │ Easier to reason about            More available│
    │ Lower availability                 Harder to use│
```

## Strong Consistency

Every read receives the most recent write or an error. Also called linearizability.

### Linearizability

Operations appear to execute atomically and in real-time order.

```
Timeline:
────────────────────────────────────────────────────────▶
         │                    │              │
    Write(X=1)           Read(X)        Read(X)
    starts              returns 1       returns 1
         │                    │              │
         │    ┌───────────────┤              │
         │    │  Write takes  │              │
         │    │  effect here  │              │
         │    │     ▼         │              │
─────────┼────┼───────────────┼──────────────┼──────────
    X=0  │    │     X=1       │              │
```

### Implementation

**Single Leader**:
```
          All writes
              │
              ▼
        ┌──────────┐
        │  Leader  │  ◀── Single point of serialization
        └────┬─────┘
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
┌───────┐┌───────┐┌───────┐
│Replica││Replica││Replica│
└───────┘└───────┘└───────┘
```

**Consensus (Paxos/Raft)**:
```
Write must be acknowledged by majority before returning
```

### Trade-offs

| Advantage | Disadvantage |
|-----------|--------------|
| Easy to reason about | Higher latency (consensus) |
| No stale reads | Lower availability (need quorum) |
| Natural semantics | Expensive across regions |

### When to Use

- Financial transactions
- Inventory management
- Leader election
- Anything requiring "read-after-write"

## Eventual Consistency

If no new updates are made, eventually all reads will return the last updated value.

```
Timeline:
────────────────────────────────────────────────────────▶
    │           │              │                   │
Write(X=1)   Read(X)       Read(X)            Read(X)
    │        returns 0     returns 1          returns 1
    │        (stale)       (maybe)            (converged)
    │           │              │                   │
    │           │              │                   │
    │    ◀──────┴──────────────┴───────────────────┤
    │         Convergence window                   │
```

### Properties

- **No staleness bound**: Reads might be arbitrarily stale
- **Eventual convergence**: All replicas converge to same value
- **High availability**: Reads/writes can always proceed

### Conflict Resolution

When concurrent writes occur:

```
Node A: Write(X=1) ──────┐
                         ├──▶ Conflict!
Node B: Write(X=2) ──────┘
```

**Resolution strategies:**

| Strategy | Description | Example |
|----------|-------------|---------|
| Last Writer Wins (LWW) | Timestamp-based | Cassandra |
| Version Vectors | Detect conflicts, let app resolve | Riak |
| CRDTs | Mathematically mergeable | Counters, sets |
| Custom logic | Application-specific | Shopping cart merge |

### CRDTs (Conflict-free Replicated Data Types)

Data structures that can be merged without conflicts:

```
G-Counter (Grow-only counter):

Node A: {A: 5, B: 0, C: 0}
Node B: {A: 0, B: 3, C: 0}
Node C: {A: 0, B: 0, C: 2}

Merge: {A: 5, B: 3, C: 2}
Total: 10
```

### When to Use

- Social media (likes, posts)
- DNS
- Caching
- Shopping carts
- User preferences

## Other Consistency Models

### Sequential Consistency

Operations appear in some sequential order, consistent across all nodes (but not necessarily real-time).

```
Client 1: Write(X=1)  Write(Y=1)
Client 2:                         Read(Y)=1  Read(X)=?

Sequential consistency allows Read(X)=0
(can reorder across clients, but each client's ops are in order)
```

### Causal Consistency

Operations that are causally related are seen in the same order by all nodes.

```
If A → B (A causes B), then everyone sees A before B

Example:
User 1: Post "Hello"
User 2: Reply "Hi" (causally depends on "Hello")

All users must see "Hello" before "Hi"
```

### Read Your Writes

A client always sees its own writes.

```
Client: Write(X=1) ─────▶ Read(X) must return 1
                              │
                              │ (even if reading from replica)
```

### Monotonic Reads

Once a value is seen, subsequent reads won't return older values.

```
Client: Read(X)=1 ─────▶ Read(X) must return ≥1
                              │
                              │ (won't see X=0)
```

### Monotonic Writes

Writes from a client are applied in order.

```
Client: Write(X=1) ─────▶ Write(X=2)

All nodes apply in this order (X=1, then X=2)
```

## Consistency in Practice

### Consistency Levels in Cassandra

```
┌─────────────┬───────────────────────────────────┬─────────────┐
│   Level     │   Description                     │   Latency   │
├─────────────┼───────────────────────────────────┼─────────────┤
│ ONE         │ 1 replica responds                │   Lowest    │
│ QUORUM      │ Majority responds (N/2 + 1)       │   Medium    │
│ ALL         │ All replicas respond              │   Highest   │
│ LOCAL_ONE   │ 1 replica in local datacenter     │   Lowest    │
│ LOCAL_QUORUM│ Quorum in local datacenter        │   Medium    │
└─────────────┴───────────────────────────────────┴─────────────┘
```

**Strong consistency in Cassandra:**
```
Write(QUORUM) + Read(QUORUM) > Replication Factor
Example: W=2 + R=2 > RF=3 ✓
```

### Consistency in DynamoDB

| Setting | Read Consistency |
|---------|------------------|
| Eventually consistent | Might read stale data |
| Strongly consistent | Reads reflect all writes |

### Tunable Consistency

```
R + W > N = Strong consistency
R + W ≤ N = Eventual consistency

R = Read quorum
W = Write quorum
N = Number of replicas

Examples (N=3):
- R=1, W=3: Slow writes, fast reads, strong
- R=3, W=1: Fast writes, slow reads, strong
- R=1, W=1: Fast, but eventual
- R=2, W=2: Balanced, strong
```

### Consistency Patterns

**Read-after-write consistency:**
```python
# Write to primary
primary.write(user_id, data)

# Option 1: Read from primary
data = primary.read(user_id)

# Option 2: Read from replica with version check
data = replica.read(user_id, min_version=write_version)

# Option 3: Sticky sessions (same user → same node)
```

**Eventual consistency with fallback:**
```python
data = cache.get(key)
if data is None:
    data = database.read(key)
    cache.set(key, data, ttl=60)
return data
```

## Key Takeaways

1. **Strong consistency** is easier to program but has performance/availability costs
2. **Eventual consistency** scales better but requires careful design
3. **Choose based on use case** - not all data needs strong consistency
4. **Understand the spectrum** - many models between strong and eventual
5. **Tunable consistency** lets you choose per-operation

## Consistency Decision Matrix

| Use Case | Recommended | Reason |
|----------|-------------|--------|
| Bank balance | Strong | Money can't be wrong |
| Like count | Eventual | Approximate is fine |
| Session data | Read-your-writes | User sees their actions |
| Chat messages | Causal | Order matters |
| DNS | Eventual | Caching is essential |
| Inventory | Strong | Prevent overselling |

## Practice Questions

1. Why can't you have strong consistency and high availability?
2. Design a shopping cart with eventual consistency.
3. How would you implement read-your-writes consistency?
4. When would you choose causal over strong consistency?

## Further Reading

- [Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter 9
- [Consistency Models](https://jepsen.io/consistency) - Jepsen
- [Life Beyond Distributed Transactions](https://queue.acm.org/detail.cfm?id=3025012)

---

Next: [CAP Theorem & PACELC](../04-cap-theorem/README.md)
