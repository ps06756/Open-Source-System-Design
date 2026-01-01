# Module 2: Core Concepts

Master the essential distributed systems concepts that appear in every system design interview. These concepts form the foundation for making architectural decisions.

## Why This Matters

Every system design question requires you to make trade-offs between:
- Consistency vs Availability
- Latency vs Throughput
- Simplicity vs Scalability

Understanding these concepts deeply will help you articulate your decisions clearly.

## Chapters

| Chapter | Topic | What You'll Learn |
|---------|-------|-------------------|
| 1 | [Scalability](01-scalability/README.md) | Horizontal vs vertical scaling, scaling strategies |
| 2 | [Availability & Reliability](02-availability/README.md) | SLAs, SLOs, fault tolerance, redundancy |
| 3 | [Consistency Models](03-consistency/README.md) | Strong, eventual, causal consistency |
| 4 | [CAP Theorem & PACELC](04-cap-theorem/README.md) | Trade-offs in distributed systems |
| 5 | [Latency vs Throughput](05-latency-throughput/README.md) | Performance metrics and optimization |
| 6 | [Data Partitioning](06-partitioning/README.md) | Sharding strategies, consistent hashing |
| 7 | [Replication](07-replication/README.md) | Leader-follower, multi-leader, leaderless |
| 8 | [Consensus](08-consensus/README.md) | Paxos, Raft, leader election |
| 9 | [Back-of-envelope Calculations](09-calculations/README.md) | QPS, storage, bandwidth estimation |

## Learning Path

```
                    ┌─────────────────┐
                    │   Scalability   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │Availability│ │Consistency │ │  Latency   │
       └──────┬─────┘ └──────┬─────┘ └──────┬─────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │   CAP Theorem   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │Partitioning│ │Replication │ │ Consensus  │
       └────────────┘ └────────────┘ └────────────┘
                             │
                    ┌────────▼────────┐
                    │  Calculations   │
                    └─────────────────┘
```

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the trade-offs between consistency, availability, and partition tolerance
2. Choose appropriate scaling strategies for different scenarios
3. Design replication and partitioning schemes
4. Perform quick capacity estimations in interviews
5. Articulate design decisions using these concepts

## Time Investment

- **First-time learning**: 8-12 hours
- **Review**: 1-2 hours

---

[← Previous Module: Fundamentals](../01-fundamentals/README.md) | [Next: Scalability →](01-scalability/README.md)

[Back to Course](../README.md)
