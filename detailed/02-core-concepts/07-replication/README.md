# Replication

Replication keeps copies of data on multiple machines for fault tolerance and performance. Understanding replication strategies is essential for designing reliable systems.

## Table of Contents
1. [Why Replicate?](#why-replicate)
2. [Leader-Follower Replication](#leader-follower-replication)
3. [Multi-Leader Replication](#multi-leader-replication)
4. [Leaderless Replication](#leaderless-replication)
5. [Replication Lag](#replication-lag)
6. [Interview Questions](#interview-questions)

---

## Why Replicate?

```
┌─────────────────────────────────────────────────────────────┐
│                  Benefits of Replication                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. High Availability                                        │
│     If one node fails, others can serve                     │
│                                                              │
│  2. Read Scalability                                         │
│     Distribute reads across replicas                        │
│                                                              │
│  3. Low Latency                                              │
│     Serve from geographically closer replica                │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │  Without replication:              With replication:  │   │
│  │                                                       │   │
│  │  ┌──────────┐                  ┌──────────┐          │   │
│  │  │  Single  │                  │  Leader  │          │   │
│  │  │ Database │                  └────┬─────┘          │   │
│  │  └──────────┘               ┌───────┼───────┐        │   │
│  │       │                     ▼       ▼       ▼        │   │
│  │       ✗ Failure          ┌────┐ ┌────┐ ┌────┐       │   │
│  │       = Data loss        │ R1 │ │ R2 │ │ R3 │       │   │
│  │       = Downtime         └────┘ └────┘ └────┘       │   │
│  │                          (Survive 2 failures)        │   │
│  │                                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Leader-Follower Replication

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│              Leader-Follower (Primary-Replica)               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  All writes go to leader, replicated to followers           │
│                                                              │
│            ┌─────────────────────────┐                      │
│            │        Leader           │                      │
│            │   (accepts writes)      │                      │
│            └───────────┬─────────────┘                      │
│                        │                                     │
│         Replication    │                                     │
│         ┌──────────────┼──────────────┐                     │
│         │              │              │                     │
│         ▼              ▼              ▼                     │
│    ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
│    │Follower1│   │Follower2│   │Follower3│                 │
│    │ (read)  │   │ (read)  │   │ (read)  │                 │
│    └─────────┘   └─────────┘   └─────────┘                 │
│                                                              │
│  Write path: Client → Leader                                │
│  Read path:  Client → Any follower (or leader)             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Synchronous vs Asynchronous

```
┌─────────────────────────────────────────────────────────────┐
│          Synchronous vs Asynchronous Replication             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Synchronous:                                                │
│  ┌────────┐     write    ┌────────┐    sync     ┌────────┐ │
│  │ Client │────────────▶│ Leader │────────────▶│Follower│ │
│  └────────┘              └────────┘             └────────┘ │
│       ▲                       │                      │      │
│       │                       │                      │      │
│       └───────────────────────┴──────────────────────┘      │
│                      wait for all ACKs                      │
│                                                              │
│  ✓ Strong consistency (no data loss)                       │
│  ✗ Higher latency                                           │
│  ✗ Unavailable if any replica down                         │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Asynchronous:                                               │
│  ┌────────┐     write    ┌────────┐   async     ┌────────┐ │
│  │ Client │────────────▶│ Leader │───────────▶│Follower│ │
│  └────────┘              └────────┘             └────────┘ │
│       ▲                       │                             │
│       └───────────────────────┘                             │
│              immediate ACK                                   │
│                                                              │
│  ✓ Low latency                                              │
│  ✓ Available even if replicas lag                          │
│  ✗ Data loss if leader fails before replication            │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Semi-synchronous:                                           │
│  Wait for at least 1 replica to ACK                        │
│  Balance between durability and performance                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Failover

```
┌─────────────────────────────────────────────────────────────┐
│                   Leader Failover                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Leader fails                                             │
│     ┌─────────┐                                             │
│     │ Leader  │ ← ✗ Crashed                                │
│     └─────────┘                                             │
│                                                              │
│  2. Followers detect failure (timeout)                      │
│     Followers stop receiving heartbeats                     │
│                                                              │
│  3. Elect new leader                                         │
│     Choose replica with most recent data                    │
│     ┌─────────┐                                             │
│     │Follower1│ ← Promoted to Leader                       │
│     └─────────┘                                             │
│                                                              │
│  4. Reconfigure system                                       │
│     Other followers connect to new leader                   │
│     Clients redirect writes to new leader                   │
│                                                              │
│  Challenges:                                                 │
│  • Split-brain (two leaders)                                │
│  • Data loss (async replication)                            │
│  • Choosing right timeout                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Multi-Leader Replication

```
┌─────────────────────────────────────────────────────────────┐
│                Multi-Leader Replication                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Multiple leaders accept writes (usually in different DCs)  │
│                                                              │
│       Datacenter A              Datacenter B                │
│  ┌─────────────────────┐  ┌─────────────────────┐          │
│  │    ┌─────────┐      │  │      ┌─────────┐    │          │
│  │    │ Leader  │◄─────┼──┼─────▶│ Leader  │    │          │
│  │    └────┬────┘      │  │      └────┬────┘    │          │
│  │         │           │  │           │         │          │
│  │    ┌────┴────┐      │  │      ┌────┴────┐    │          │
│  │    │Follower │      │  │      │Follower │    │          │
│  │    └─────────┘      │  │      └─────────┘    │          │
│  └─────────────────────┘  └─────────────────────┘          │
│                                                              │
│  Use cases:                                                  │
│  • Multi-datacenter deployment                              │
│  • Offline clients (mobile, laptop)                        │
│                                                              │
│  Challenges:                                                 │
│  • Write conflicts!                                          │
│    User edits doc in DC-A, another in DC-B                  │
│  • Conflict resolution needed                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Conflict Resolution

```
┌─────────────────────────────────────────────────────────────┐
│                  Conflict Resolution                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Two users update same record simultaneously:               │
│                                                              │
│  User A (DC-A): title = "Hello"                             │
│  User B (DC-B): title = "World"                             │
│                                                              │
│  Resolution strategies:                                      │
│                                                              │
│  1. Last-Write-Wins (LWW)                                   │
│     Use timestamp, keep most recent                         │
│     Simple but loses data                                   │
│                                                              │
│  2. Merge values                                             │
│     Combine: title = "Hello World"                          │
│     Works for some data types                               │
│                                                              │
│  3. Keep both (siblings)                                    │
│     Return both to application                              │
│     Let user decide                                         │
│                                                              │
│  4. Custom logic                                             │
│     Application-specific resolution                         │
│     e.g., higher priority user wins                         │
│                                                              │
│  5. CRDTs                                                    │
│     Conflict-free Replicated Data Types                     │
│     Mathematically guaranteed to converge                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Leaderless Replication

```
┌─────────────────────────────────────────────────────────────┐
│                 Leaderless Replication                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  No designated leader - any node accepts writes             │
│                                                              │
│         ┌─────────┐                                         │
│         │  Client │                                         │
│         └────┬────┘                                         │
│              │                                               │
│    ┌─────────┼─────────┐                                    │
│    │         │         │                                    │
│    ▼         ▼         ▼                                    │
│  ┌───┐    ┌───┐     ┌───┐                                  │
│  │ A │    │ B │     │ C │                                  │
│  └───┘    └───┘     └───┘                                  │
│                                                              │
│  Write: Send to all replicas (or W of N)                   │
│  Read:  Read from all replicas (or R of N)                 │
│         Return most recent version                          │
│                                                              │
│  Used by: Cassandra, DynamoDB, Riak                        │
│                                                              │
│  Quorum: W + R > N guarantees reading latest write         │
│  Example: N=3, W=2, R=2                                    │
│           Write to 2, Read from 2                           │
│           At least 1 node has latest                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Replication Lag

```
┌─────────────────────────────────────────────────────────────┐
│                    Replication Lag                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Time between write on leader and visible on follower       │
│                                                              │
│  Leader:   ───[Write]──────────────────────────────────▶   │
│                 │                                            │
│  Follower: ─────┼────────[Applied]─────────────────────▶   │
│                 │◄───────────────▶│                         │
│                    Replication Lag                          │
│                                                              │
│  Problems caused by lag:                                     │
│                                                              │
│  1. Reading your writes                                      │
│     Write profile, refresh, see OLD profile                 │
│     Solution: Read from leader after write                  │
│                                                              │
│  2. Monotonic reads                                          │
│     First read: 10 comments                                 │
│     Second read: 8 comments (different replica)             │
│     Solution: Sticky sessions (same replica)               │
│                                                              │
│  3. Consistent prefix reads                                  │
│     See answer before question                              │
│     Solution: Causal ordering                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is replication and why is it important?
2. Explain leader-follower replication.
3. What is the difference between sync and async replication?

### Intermediate
4. How does leader failover work? What are the challenges?
5. When would you use multi-leader replication?
6. What is a quorum and how does it ensure consistency?

### Advanced
7. How would you handle replication lag in a read-heavy system?
8. Design a multi-datacenter database with conflict resolution.
9. Compare leader-follower, multi-leader, and leaderless replication.

---

[← Previous: Partitioning](../06-partitioning/README.md) | [Next: Consensus →](../08-consensus/README.md)
