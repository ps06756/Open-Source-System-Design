# Consistency Models

Consistency determines how and when updates to data become visible to different parts of a distributed system. Understanding consistency trade-offs is essential for system design.

## Table of Contents
1. [What is Consistency?](#what-is-consistency)
2. [Consistency Models](#consistency-models)
3. [Consistency in Practice](#consistency-in-practice)
4. [Interview Questions](#interview-questions)

---

## What is Consistency?

```
┌─────────────────────────────────────────────────────────────┐
│                  The Consistency Problem                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User A writes: balance = $100                              │
│                                                              │
│        ┌─────────────┐                                      │
│        │   Primary   │                                      │
│        │ balance=$100│                                      │
│        └──────┬──────┘                                      │
│               │ replication (takes time)                    │
│         ┌─────┴─────┐                                       │
│         ▼           ▼                                       │
│  ┌──────────┐ ┌──────────┐                                  │
│  │ Replica 1│ │ Replica 2│                                  │
│  │   $100   │ │   $50    │ ← still has old value            │
│  └──────────┘ └──────────┘                                  │
│                                                              │
│  User B reads from Replica 2: sees $50 (stale!)             │
│                                                              │
│  Question: When should User B see the new value?            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Consistency Models

### Strong Consistency (Linearizability)

```
┌─────────────────────────────────────────────────────────────┐
│                  Strong Consistency                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "Every read receives the most recent write"                │
│                                                              │
│  Timeline:                                                   │
│  ─────────────────────────────────────────────────▶ time    │
│       │                   │                   │              │
│   Write(x=1)          Read(x)             Read(x)           │
│       │                   │                   │              │
│       └───────────────────┼───────────────────┘              │
│                           │                                  │
│                    Both return 1                            │
│                                                              │
│  Implementation: Synchronous replication                    │
│                                                              │
│  ┌─────────┐     write     ┌─────────┐                     │
│  │ Client  │──────────────▶│ Primary │                     │
│  └─────────┘               └────┬────┘                     │
│       ▲                         │ sync replicate           │
│       │                    ┌────┴────┐                      │
│       │                    ▼         ▼                      │
│       │              ┌─────────┐ ┌─────────┐               │
│       │              │Replica 1│ │Replica 2│               │
│       │              └────┬────┘ └────┬────┘               │
│       │                   │ ACK       │ ACK                 │
│       │                   └─────┬─────┘                     │
│       └───── OK ────────────────┘                           │
│                (only after ALL replicas confirm)            │
│                                                              │
│  Pros: Simplest programming model                           │
│  Cons: High latency, lower availability                    │
│                                                              │
│  Examples: Traditional RDBMS, ZooKeeper, etcd              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Eventual Consistency

```
┌─────────────────────────────────────────────────────────────┐
│                  Eventual Consistency                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "If no new updates, all replicas will eventually           │
│   converge to the same value"                               │
│                                                              │
│  Timeline:                                                   │
│  ─────────────────────────────────────────────────▶ time    │
│       │           │           │           │                  │
│   Write(x=1)  Read(x)     Read(x)     Read(x)               │
│       │           │           │           │                  │
│               returns 0   returns 0   returns 1             │
│               (stale)     (stale)     (consistent)          │
│                                                              │
│  Implementation: Asynchronous replication                   │
│                                                              │
│  ┌─────────┐     write     ┌─────────┐                     │
│  │ Client  │──────────────▶│ Primary │                     │
│  └─────────┘               └────┬────┘                     │
│       ▲                         │                           │
│       └──── OK ─────────────────┘  (returns immediately)   │
│                                    │                        │
│                              async │ replicate              │
│                               ┌────┴────┐                   │
│                               ▼         ▼                   │
│                         ┌─────────┐ ┌─────────┐            │
│                         │Replica 1│ │Replica 2│            │
│                         └─────────┘ └─────────┘            │
│                         (updates arrive eventually)         │
│                                                              │
│  Pros: Low latency, high availability                       │
│  Cons: May read stale data                                  │
│                                                              │
│  Examples: Cassandra, DynamoDB, DNS                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Causal Consistency

```
┌─────────────────────────────────────────────────────────────┐
│                   Causal Consistency                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "Causally related operations are seen in order"            │
│                                                              │
│  Example:                                                    │
│  User A posts: "What's for lunch?"                          │
│  User B replies: "Pizza!"                                   │
│                                                              │
│  Causal order: Post MUST be seen before Reply              │
│  (B's reply was caused by A's post)                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  User A: Posts question                             │   │
│  │     │                                                │   │
│  │     └──▶ User B: Sees question, posts reply         │   │
│  │              │                                       │   │
│  │              └──▶ User C must see:                  │   │
│  │                   1. Question first                 │   │
│  │                   2. Then reply                     │   │
│  │                                                      │   │
│  │  Unrelated posts (not causal) can appear in any    │   │
│  │  order across different users.                      │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Implementation: Vector clocks, version vectors            │
│                                                              │
│  Pros: Intuitive for users, better than eventual           │
│  Cons: Complex implementation                              │
│                                                              │
│  Examples: MongoDB (with causal sessions), COPS            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Read-Your-Writes Consistency

```
┌─────────────────────────────────────────────────────────────┐
│               Read-Your-Writes Consistency                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "A user always sees their own writes"                      │
│                                                              │
│  User A writes profile update                               │
│  User A refreshes page                                      │
│  User A MUST see their update (not stale data)             │
│                                                              │
│  Implementation options:                                     │
│                                                              │
│  1. Read from primary after write                          │
│     Write → Primary → Read from Primary                    │
│                                                              │
│  2. Track write timestamp                                   │
│     If replica < write_timestamp, read from primary        │
│                                                              │
│  3. Sticky sessions                                         │
│     Route user to same replica always                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  User writes to Primary                               │  │
│  │       │                                               │  │
│  │       ▼                                               │  │
│  │  Store write_timestamp = T1                          │  │
│  │       │                                               │  │
│  │       ▼                                               │  │
│  │  User reads                                           │  │
│  │       │                                               │  │
│  │       ├─▶ If replica.version >= T1 → read replica    │  │
│  │       │                                               │  │
│  │       └─▶ Else → read primary                        │  │
│  │                                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Comparison

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Consistency Model Comparison                          │
├────────────────────┬──────────────┬──────────────┬────────────────────────┤
│ Model              │ Guarantee    │ Performance  │ Use Case               │
├────────────────────┼──────────────┼──────────────┼────────────────────────┤
│ Strong             │ Always fresh │ Slow         │ Banking, inventory     │
│ Eventual           │ Eventually   │ Fast         │ Social feeds, likes    │
│ Causal             │ Ordered      │ Medium       │ Collaboration, chat    │
│ Read-Your-Writes   │ Own writes   │ Fast         │ User profiles          │
└────────────────────┴──────────────┴──────────────┴────────────────────────┘
```

---

## Consistency in Practice

### Conflict Resolution

```
┌─────────────────────────────────────────────────────────────┐
│                  Conflict Resolution                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Two users update same data simultaneously:                 │
│                                                              │
│  User A: Set title = "Hello"                                │
│  User B: Set title = "World"                                │
│                                                              │
│  Resolution strategies:                                      │
│                                                              │
│  1. Last-Write-Wins (LWW)                                   │
│     Use timestamp, keep most recent                         │
│     Simple but can lose updates                             │
│                                                              │
│  2. First-Write-Wins                                         │
│     Reject later writes                                      │
│     Used for immutable data                                  │
│                                                              │
│  3. Merge/CRDT                                               │
│     Combine both values automatically                        │
│     Works for sets, counters, lists                         │
│                                                              │
│  4. Application Resolution                                   │
│     Return both versions to app                             │
│     App/user decides (like Git conflicts)                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Quorum Reads/Writes

```
┌─────────────────────────────────────────────────────────────┐
│                      Quorum System                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  N = Total replicas                                          │
│  W = Write quorum (replicas that must acknowledge write)    │
│  R = Read quorum (replicas that must respond to read)       │
│                                                              │
│  Strong consistency if: W + R > N                           │
│                                                              │
│  Example: N=3, W=2, R=2                                     │
│                                                              │
│  Write to 2 of 3:          Read from 2 of 3:               │
│  ┌───┐ ┌───┐ ┌───┐         ┌───┐ ┌───┐ ┌───┐               │
│  │ ✓ │ │ ✓ │ │   │         │ v1│ │ v2│ │   │               │
│  └───┘ └───┘ └───┘         └───┘ └───┘ └───┘               │
│    A     B     C             A     B     C                  │
│                                                              │
│  At least one overlap → guaranteed to see latest           │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Common configurations:                               │   │
│  │                                                      │   │
│  │ W=N, R=1:   Strong writes, fast reads               │   │
│  │ W=1, R=N:   Fast writes, strong reads               │   │
│  │ W=N/2+1, R=N/2+1: Balanced                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between strong and eventual consistency?
2. Give an example where eventual consistency is acceptable.
3. What is read-your-writes consistency?

### Intermediate
4. Explain quorum reads and writes. What values ensure strong consistency?
5. How would you implement causal consistency?
6. What are the trade-offs between different consistency levels?

### Advanced
7. Design a shopping cart that handles concurrent updates.
8. How does Cassandra achieve tunable consistency?
9. Explain how vector clocks help with conflict detection.

---

[← Previous: Availability](../02-availability/README.md) | [Next: CAP Theorem →](../04-cap-theorem/README.md)
