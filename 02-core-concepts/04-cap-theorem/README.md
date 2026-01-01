# CAP Theorem & PACELC

The CAP theorem and PACELC are fundamental frameworks for understanding trade-offs in distributed systems.

## Table of Contents
- [CAP Theorem](#cap-theorem)
- [Understanding the Trade-offs](#understanding-the-trade-offs)
- [PACELC Theorem](#pacelc-theorem)
- [Real-World Systems](#real-world-systems)
- [Key Takeaways](#key-takeaways)

## CAP Theorem

The CAP theorem states that a distributed system can only provide two of the following three guarantees:

```
                    Consistency (C)
                         /\
                        /  \
                       /    \
                      /  CP  \
                     /        \
                    /──────────\
                   /            \
                  /      CA      \
                 /    (single     \
                /      node)       \
               /────────────────────\
              /          AP          \
             ────────────────────────
     Availability (A)            Partition
                                 Tolerance (P)
```

### Definitions

**Consistency (C)**: Every read receives the most recent write or an error.

**Availability (A)**: Every request receives a response (success or failure), without guarantee it's the most recent write.

**Partition Tolerance (P)**: The system continues to operate despite network partitions between nodes.

### Why You Can Only Choose Two

In a distributed system, network partitions **will happen**. So you must choose P.

```
Network Partition:

┌─────────────┐          ╳          ┌─────────────┐
│   Node A    │     (partition)     │   Node B    │
│   X = 1     │                     │   X = 1     │
└─────────────┘                     └─────────────┘

Client writes X = 2 to Node A...

┌─────────────┐          ╳          ┌─────────────┐
│   Node A    │     (partition)     │   Node B    │
│   X = 2     │                     │   X = 1     │
└─────────────┘                     └─────────────┘

Now a client reads from Node B...
```

**Choice 1 - CP (Consistency over Availability):**
```
Node B: "Sorry, I can't respond because I might be stale"
         (Sacrifice availability)
```

**Choice 2 - AP (Availability over Consistency):**
```
Node B: "Here's X = 1" (might be stale)
         (Sacrifice consistency)
```

### CP Systems

Choose consistency over availability during partitions.

```
During partition:
┌─────────────┐          ╳          ┌─────────────┐
│   Node A    │                     │   Node B    │
│   (active)  │                     │   (blocked) │
└─────────────┘                     └─────────────┘

- Writes rejected on minority side
- Reads may be rejected if can't confirm freshness
```

**Examples:**
- MongoDB (with majority read/write concern)
- HBase
- Redis Cluster (partial)
- Zookeeper
- etcd

**Use when:**
- Data correctness is critical
- Financial transactions
- Inventory systems
- Configuration management

### AP Systems

Choose availability over consistency during partitions.

```
During partition:
┌─────────────┐          ╳          ┌─────────────┐
│   Node A    │                     │   Node B    │
│  (accepts   │                     │  (accepts   │
│   writes)   │                     │   writes)   │
└─────────────┘                     └─────────────┘

- Both sides accept reads and writes
- Conflicts resolved after partition heals
```

**Examples:**
- Cassandra
- DynamoDB
- CouchDB
- Riak

**Use when:**
- Availability is critical
- Eventual consistency is acceptable
- Social media, shopping carts
- DNS

## Understanding the Trade-offs

### What CAP Doesn't Tell You

1. **It's not all-or-nothing**: You can tune consistency per-operation
2. **Latency matters**: CAP ignores the cost of consistency
3. **Partitions are rare**: What matters day-to-day is latency vs consistency

### Common Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Choose 2 of 3" | You must choose P, then decide C vs A |
| "CA is possible" | Only with a single node (not distributed) |
| "Binary choice" | Consistency is a spectrum |
| "Always matters" | Trade-off only during partitions |

## PACELC Theorem

PACELC extends CAP to address normal operation (no partition).

```
if (Partition) {
    choose Availability or Consistency  // CAP
} else {
    choose Latency or Consistency       // ELC
}
```

### The Full Picture

```
┌─────────────────────────────────────────────────────────┐
│                      PACELC                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   During Partition:          Normal Operation:          │
│   ┌─────────────┐           ┌─────────────────┐        │
│   │ A or C?     │           │ L or C?         │        │
│   │             │           │                 │        │
│   │ PA: Avail   │           │ EL: Low Latency │        │
│   │ PC: Consist │           │ EC: Consistency │        │
│   └─────────────┘           └─────────────────┘        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### PACELC Categories

| Category | During Partition | Normal Operation | Example |
|----------|------------------|------------------|---------|
| PA/EL | Availability | Low Latency | Cassandra, DynamoDB |
| PA/EC | Availability | Consistency | ? (rare) |
| PC/EL | Consistency | Low Latency | ? (rare) |
| PC/EC | Consistency | Consistency | MongoDB, HBase |

### Why PACELC Matters

Most of the time there's no partition. PACELC helps you think about:
- What's the latency cost of consistency?
- Is strong consistency worth the extra RTT?

```
Strong Consistency (EC):
Client ──▶ Leader ──▶ Wait for majority ──▶ Response
                         (extra latency)

Eventual Consistency (EL):
Client ──▶ Any Node ──▶ Response
              (immediate)
```

## Real-World Systems

### DynamoDB

**Classification: PA/EL**
- During partition: Stays available, eventually consistent
- Normal operation: Fast reads (eventual), optional strong reads

```
┌────────────────────────────────────────────────────────┐
│                    DynamoDB                            │
├────────────────────────────────────────────────────────┤
│  Eventual read:  Low latency, might be stale          │
│  Strong read:    Higher latency, always fresh         │
│  Write:          Returns after local commit           │
└────────────────────────────────────────────────────────┘
```

### Cassandra

**Classification: PA/EL (tunable)**
- During partition: Stays available
- Normal operation: Tunable via consistency levels

```
┌────────────────────────────────────────────────────────┐
│                    Cassandra                           │
├────────────────────────────────────────────────────────┤
│  CL=ONE:     PA/EL - Fast, eventually consistent      │
│  CL=QUORUM:  PC/EC - Slower, strongly consistent      │
│  CL=ALL:     PC/EC - Slowest, maximum consistency     │
└────────────────────────────────────────────────────────┘
```

### MongoDB

**Classification: PC/EC (with majority concerns)**
- During partition: Minority becomes unavailable
- Normal operation: Reads from primary (consistent)

```
┌────────────────────────────────────────────────────────┐
│                     MongoDB                            │
├────────────────────────────────────────────────────────┤
│  Primary reads:      Strong consistency               │
│  Secondary reads:    Eventual consistency             │
│  Write concern=maj:  Durable + consistent             │
└────────────────────────────────────────────────────────┘
```

### Spanner (Google)

**Classification: PC/EC**
- Uses TrueTime for global strong consistency
- Accepts latency cost (~14ms+ for writes)

```
┌────────────────────────────────────────────────────────┐
│                     Spanner                            │
├────────────────────────────────────────────────────────┤
│  External consistency (linearizability)               │
│  Synchronous replication across regions               │
│  TrueTime: GPS + atomic clocks for ordering           │
│  Trade-off: Higher latency for global consistency     │
└────────────────────────────────────────────────────────┘
```

### Summary Table

| System | CAP | PACELC | Notes |
|--------|-----|--------|-------|
| Cassandra | AP | PA/EL | Tunable consistency |
| DynamoDB | AP | PA/EL | Eventual default, strong option |
| MongoDB | CP | PC/EC | With majority concerns |
| PostgreSQL | CA* | N/A | Single node = no partition |
| CockroachDB | CP | PC/EC | Serializable, distributed |
| Redis | CP | PC/EL | Cluster mode |
| Zookeeper | CP | PC/EC | Consensus-based |
| Riak | AP | PA/EL | CRDTs for conflict resolution |

*Single-node PostgreSQL is CA, but that's not a distributed system

## Design Implications

### Choosing Your Trade-off

```
┌─────────────────────────────────────────────────────────┐
│           Use Case → Trade-off Decision                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Banking/Payments     →  PC/EC (correctness critical)  │
│  Social Media Likes   →  PA/EL (speed over accuracy)   │
│  E-commerce Cart      →  PA/EL (merge conflicts later) │
│  Inventory Count      →  PC/EC (avoid overselling)     │
│  User Sessions        →  PA/EL (availability critical) │
│  Leader Election      →  PC/EC (exactly one leader)    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Hybrid Approaches

Different data, different trade-offs:

```
┌─────────────────────────────────────────────────────────┐
│              E-commerce Platform                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Product Catalog    →  AP (cache heavily, stale OK)    │
│  Shopping Cart      →  AP (merge on checkout)          │
│  Inventory          →  CP (accurate counts needed)     │
│  Order Placement    →  CP (transactional)              │
│  Order History      →  AP (eventual consistency fine)  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **CAP is about partitions** - you're choosing between C and A during failures
2. **PACELC is about latency** - even without partitions, consistency has a cost
3. **Most systems are PA/EL** - because availability and speed are usually preferred
4. **It's not binary** - you can tune consistency per-operation
5. **Choose based on data** - different data has different requirements

## Practice Questions

1. You're designing a global messaging app. What CAP trade-off would you make and why?
2. Explain why a banking system would choose CP over AP.
3. How does Cassandra achieve tunable consistency?
4. What's the practical difference between PA/EL and PC/EC systems?

## Further Reading

- [CAP Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)
- [PACELC on Wikipedia](https://en.wikipedia.org/wiki/PACELC_theorem)
- [Jepsen: Consistency Models](https://jepsen.io/consistency)

---

Next: [Latency vs Throughput](../05-latency-throughput/README.md)
