# Consensus

Consensus algorithms allow distributed systems to agree on values even when some nodes fail. Understanding consensus is crucial for building reliable distributed systems.

## Table of Contents
1. [The Consensus Problem](#the-consensus-problem)
2. [Leader Election](#leader-election)
3. [Raft Algorithm](#raft-algorithm)
4. [Practical Applications](#practical-applications)
5. [Interview Questions](#interview-questions)

---

## The Consensus Problem

```
┌─────────────────────────────────────────────────────────────┐
│                  The Consensus Problem                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  How do distributed nodes agree on a single value?          │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                                                        │  │
│  │    Node A         Node B         Node C               │  │
│  │    ┌───┐          ┌───┐          ┌───┐                │  │
│  │    │ 5 │          │ 7 │          │ 5 │                │  │
│  │    └───┘          └───┘          └───┘                │  │
│  │                                                        │  │
│  │    All must agree on ONE value (5 or 7)               │  │
│  │                                                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Requirements:                                               │
│                                                              │
│  1. Agreement: All non-faulty nodes decide same value       │
│  2. Validity: Decided value was proposed by some node       │
│  3. Termination: All non-faulty nodes eventually decide     │
│  4. Integrity: Nodes decide only once                       │
│                                                              │
│  Challenges:                                                 │
│  • Network delays and partitions                            │
│  • Node crashes                                              │
│  • Byzantine (malicious) nodes                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Leader Election

```
┌─────────────────────────────────────────────────────────────┐
│                    Leader Election                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Most consensus algorithms elect a leader to coordinate     │
│                                                              │
│  Why a leader?                                               │
│  • Simplifies coordination                                  │
│  • Avoids conflicts                                          │
│  • Orders operations                                         │
│                                                              │
│  Election process (simplified):                             │
│                                                              │
│  1. Leader sends heartbeats                                  │
│     ┌────────┐                                              │
│     │ Leader │ ─── heartbeat ───▶ followers                │
│     └────────┘                                              │
│                                                              │
│  2. Followers detect leader failure (no heartbeat)          │
│     Timeout! Leader might be dead                           │
│                                                              │
│  3. Candidate starts election                                │
│     ┌───────────┐                                           │
│     │ Candidate │ ─── "Vote for me!" ───▶                  │
│     └───────────┘                                           │
│                                                              │
│  4. Majority votes → new leader                             │
│     3 of 5 nodes vote yes → elected                        │
│                                                              │
│  5. New leader starts sending heartbeats                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Raft Algorithm

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Raft Overview                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Designed for understandability (easier than Paxos)        │
│                                                              │
│  Node states:                                                │
│  ┌──────────┐   timeout    ┌───────────┐   majority       │
│  │ Follower │ ───────────▶ │ Candidate │ ───────────▶     │
│  └──────────┘              └───────────┘                   │
│       ▲                          │                          │
│       │                          │ lose                     │
│       │                          ▼                          │
│       │                    ┌──────────┐                    │
│       └────────────────────│  Leader  │                    │
│         (new leader wins)  └──────────┘                    │
│                                                              │
│  Key concepts:                                               │
│  • Term: Logical clock, increments on elections            │
│  • Log: Ordered list of commands                           │
│  • Commitment: Entry replicated to majority                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Leader Election in Raft

```
┌─────────────────────────────────────────────────────────────┐
│               Raft Leader Election                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Term 1:                                                     │
│  ┌──────────┐  heartbeat  ┌──────────┐  ┌──────────┐       │
│  │ Leader A │ ──────────▶ │Follower B│  │Follower C│       │
│  └──────────┘             └──────────┘  └──────────┘       │
│                                                              │
│  Leader A crashes...                                         │
│                                                              │
│  Term 2:                                                     │
│  ┌──────────┐             ┌──────────┐  ┌──────────┐       │
│  │  Dead A  │             │Candidate B│  │Follower C│       │
│  └──────────┘             └──────────┘  └──────────┘       │
│                                 │                           │
│                    ┌────────────┴────────────┐             │
│                    │   RequestVote RPC       │             │
│                    │   "Term 2, vote for B"  │             │
│                    └────────────┬────────────┘             │
│                                 ▼                           │
│                           C votes for B                     │
│                           B votes for self                  │
│                           B has majority (2/3)              │
│                                                              │
│  Term 2:                                                     │
│  ┌──────────┐             ┌──────────┐  ┌──────────┐       │
│  │  Dead A  │             │ Leader B │  │Follower C│       │
│  └──────────┘             └──────────┘  └──────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Log Replication in Raft

```
┌─────────────────────────────────────────────────────────────┐
│                  Log Replication                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Client sends command to leader                          │
│                                                              │
│  2. Leader appends to its log                               │
│     ┌─────┬─────┬─────┬─────┐                              │
│     │ x=1 │ y=2 │ x=3 │ NEW │  ← Leader's log              │
│     └─────┴─────┴─────┴─────┘                              │
│                                                              │
│  3. Leader sends AppendEntries to followers                 │
│                                                              │
│  4. Followers append to their logs                          │
│     ┌─────┬─────┬─────┬─────┐                              │
│     │ x=1 │ y=2 │ x=3 │ NEW │  ← Follower's log            │
│     └─────┴─────┴─────┴─────┘                              │
│                                                              │
│  5. Majority acknowledged → entry committed                 │
│                                                              │
│  6. Leader applies to state machine                         │
│                                                              │
│  7. Leader notifies followers of commit                     │
│                                                              │
│  8. Followers apply to their state machines                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Safety

```
┌─────────────────────────────────────────────────────────────┐
│                    Raft Safety                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Election Restriction:                                       │
│  Candidate must have all committed entries                  │
│  Voter rejects candidates with outdated log                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                                                       │  │
│  │  Candidate's log:  [1, 2, 3]                         │  │
│  │  Voter's log:      [1, 2, 3, 4, 5]                   │  │
│  │                                                       │  │
│  │  Voter rejects! Candidate is behind.                 │  │
│  │                                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  This ensures:                                               │
│  • New leader has all committed entries                    │
│  • No data loss during leader changes                      │
│  • Committed entries are never overwritten                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Practical Applications

```
┌─────────────────────────────────────────────────────────────┐
│             Where Consensus is Used                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Coordination Services                                    │
│     • ZooKeeper (ZAB protocol)                              │
│     • etcd (Raft)                                           │
│     • Consul (Raft)                                         │
│     Use: Service discovery, config, leader election        │
│                                                              │
│  2. Distributed Databases                                    │
│     • CockroachDB (Raft)                                    │
│     • TiDB (Raft)                                           │
│     • Spanner (Paxos)                                       │
│     Use: Consistent replication                             │
│                                                              │
│  3. Distributed File Systems                                 │
│     • HDFS NameNode (ZooKeeper)                             │
│     Use: Metadata consistency                               │
│                                                              │
│  4. Message Queues                                           │
│     • Kafka (ZooKeeper → KRaft)                             │
│     Use: Partition leader election                          │
│                                                              │
│  NOT used for:                                               │
│  • Every write (too slow)                                   │
│  • Eventually consistent systems                            │
│  • High-throughput data storage                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the consensus problem?
2. Why do distributed systems need leader election?
3. What happens when a leader fails in Raft?

### Intermediate
4. Explain the Raft leader election process.
5. How does Raft ensure log consistency?
6. What is a term in Raft and why is it important?

### Advanced
7. How does Raft handle network partitions?
8. Compare Raft and Paxos. Why was Raft designed?
9. Design a distributed lock using consensus.

---

[← Previous: Replication](../07-replication/README.md) | [Next: Calculations →](../09-calculations/README.md)
