# CAP Theorem & PACELC

The CAP theorem is fundamental to understanding trade-offs in distributed systems. Every system design interview involves these concepts.

## Table of Contents
1. [CAP Theorem](#cap-theorem)
2. [PACELC Extension](#pacelc-extension)
3. [Real-World Examples](#real-world-examples)
4. [Interview Questions](#interview-questions)

---

## CAP Theorem

### The Three Properties

```
┌─────────────────────────────────────────────────────────────┐
│                      CAP Theorem                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  In a distributed system, you can only guarantee            │
│  TWO of these THREE properties:                             │
│                                                              │
│                      Consistency                             │
│                          /\                                  │
│                         /  \                                 │
│                        /    \                                │
│                       /      \                               │
│                      /   CA   \                              │
│                     /          \                             │
│                    /____________\                            │
│        Availability ─── CP  AP ─── Partition                │
│                                    Tolerance                 │
│                                                              │
│  C - Consistency: Every read receives the most recent write │
│  A - Availability: Every request receives a response        │
│  P - Partition Tolerance: System works despite network      │
│                           partitions                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why Choose Two?

```
┌─────────────────────────────────────────────────────────────┐
│              Network Partition Scenario                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Normal operation:                                           │
│  ┌──────────┐    network    ┌──────────┐                    │
│  │  Node A  │◄────────────►│  Node B  │                    │
│  │  Data=1  │              │  Data=1  │                    │
│  └──────────┘              └──────────┘                    │
│                                                              │
│  During partition (network split):                          │
│  ┌──────────┐      ✗ ✗     ┌──────────┐                    │
│  │  Node A  │    ✗     ✗   │  Node B  │                    │
│  │  Data=?  │   ✗   ✗   ✗  │  Data=?  │                    │
│  └──────────┘    ✗     ✗   └──────────┘                    │
│       │                          │                          │
│       │ Client writes Data=2     │ Client reads Data       │
│       ▼                          ▼                          │
│                                                              │
│  Choice 1: Consistency (CP)                                 │
│  - Reject the write or read until partition heals          │
│  - Sacrifice: Availability                                  │
│                                                              │
│  Choice 2: Availability (AP)                                │
│  - Accept write on A, return stale data on B               │
│  - Sacrifice: Consistency                                   │
│                                                              │
│  You CANNOT choose CA in a distributed system because       │
│  network partitions WILL happen. P is mandatory.           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### CP vs AP Systems

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CP vs AP Systems                                  │
├────────────────────────────────────┬─────────────────────────────────────┤
│           CP Systems               │           AP Systems                │
├────────────────────────────────────┼─────────────────────────────────────┤
│                                    │                                      │
│  Prioritize: Consistency           │  Prioritize: Availability           │
│                                    │                                      │
│  During partition:                 │  During partition:                  │
│  - Reject requests                 │  - Serve potentially stale data    │
│  - Wait for consensus              │  - Accept writes locally            │
│  - Return error to client          │  - Reconcile later                  │
│                                    │                                      │
│  Use when:                         │  Use when:                          │
│  - Data accuracy critical          │  - Uptime critical                  │
│  - Financial transactions          │  - User experience priority         │
│  - Inventory management            │  - Social media feeds               │
│                                    │                                      │
│  Examples:                         │  Examples:                          │
│  - MongoDB (in some configs)       │  - Cassandra                        │
│  - HBase                           │  - DynamoDB                         │
│  - Redis Cluster                   │  - CouchDB                          │
│  - etcd, ZooKeeper                 │  - Riak                             │
│                                    │                                      │
└────────────────────────────────────┴─────────────────────────────────────┘
```

---

## PACELC Extension

### Beyond CAP

```
┌─────────────────────────────────────────────────────────────┐
│                        PACELC                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  "If there is a Partition (P), choose between               │
│   Availability (A) and Consistency (C);                     │
│   Else (E), when running normally, choose between           │
│   Latency (L) and Consistency (C)"                          │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │         Is there a network partition?               │   │
│  │                    │                                 │   │
│  │          ┌─────────┴─────────┐                      │   │
│  │          ▼                   ▼                      │   │
│  │         Yes                  No                     │   │
│  │          │                   │                      │   │
│  │    ┌─────┴─────┐       ┌─────┴─────┐               │   │
│  │    ▼           ▼       ▼           ▼               │   │
│  │   PA          PC      EL          EC               │   │
│  │ (Avail)   (Consist)  (Latency) (Consist)           │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Full notation: PA/EL, PC/EC, PA/EC, PC/EL                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### System Classifications

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     PACELC Classifications                                │
├─────────────┬─────────────────┬──────────────────────────────────────────┤
│ System      │ Classification  │ Explanation                              │
├─────────────┼─────────────────┼──────────────────────────────────────────┤
│ Cassandra   │ PA/EL           │ Available during partition,              │
│ DynamoDB    │                 │ low latency during normal ops            │
├─────────────┼─────────────────┼──────────────────────────────────────────┤
│ MongoDB     │ PC/EC           │ Consistent always, pays latency          │
│ (default)   │                 │ cost for synchronous replication         │
├─────────────┼─────────────────┼──────────────────────────────────────────┤
│ PNUTS       │ PC/EL           │ Consistent during partition,             │
│ (Yahoo)     │                 │ low latency normally                     │
├─────────────┼─────────────────┼──────────────────────────────────────────┤
│ VoltDB      │ PC/EC           │ Strong consistency always                │
├─────────────┴─────────────────┴──────────────────────────────────────────┤
│                                                                           │
│  Most real systems offer tunable consistency:                            │
│  - Cassandra: Adjust via quorum settings                                 │
│  - MongoDB: Read/write concern levels                                    │
│  - DynamoDB: Eventually vs strongly consistent reads                     │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Real-World Examples

### Banking System (CP)

```
┌─────────────────────────────────────────────────────────────┐
│                Banking System (CP Choice)                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirement: Never show wrong balance                       │
│                                                              │
│  Scenario: User transfers $100                               │
│                                                              │
│  ┌──────────┐          ┌──────────┐                         │
│  │  Bank A  │    ✗     │  Bank B  │  (network partition)   │
│  │ Balance: │          │ Balance: │                         │
│  │  $1000   │          │  $1000   │                         │
│  └──────────┘          └──────────┘                         │
│                                                              │
│  CP Choice: Block transaction until network restored        │
│             Better to fail than to double-spend             │
│                                                              │
│  User experience: "Transaction pending, please wait"        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Social Media Feed (AP)

```
┌─────────────────────────────────────────────────────────────┐
│               Social Media Feed (AP Choice)                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirement: Always show something                         │
│                                                              │
│  Scenario: User views feed during partition                 │
│                                                              │
│  ┌──────────┐          ┌──────────┐                         │
│  │  Feed    │    ✗     │  Feed    │  (network partition)   │
│  │Server US │          │Server EU │                         │
│  │ 100 posts│          │ 98 posts │                         │
│  └──────────┘          └──────────┘                         │
│                                                              │
│  AP Choice: Show available posts, even if slightly stale    │
│             Missing 2 posts is better than error page       │
│                                                              │
│  User experience: Normal feed, maybe missing latest post    │
│                   (user probably won't notice)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Shopping Cart (Hybrid)

```
┌─────────────────────────────────────────────────────────────┐
│                 Shopping Cart (Hybrid)                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Different operations need different guarantees:            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Add to cart:     AP (always accept, merge later)       │ │
│  │ View cart:       AP (show what we have)                │ │
│  │ Checkout:        CP (must have correct inventory)      │ │
│  │ Payment:         CP (must be transactional)            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Amazon's approach:                                          │
│  - Cart uses AP (DynamoDB)                                  │
│  - Checkout verifies with CP system                         │
│  - "Sorry, item out of stock" at checkout is acceptable    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. Explain the CAP theorem in simple terms.
2. Why can't we have all three: C, A, and P?
3. Give an example of a CP system and an AP system.

### Intermediate
4. What is PACELC and how does it extend CAP?
5. For a banking application, would you choose CP or AP? Why?
6. How does Cassandra allow you to tune consistency?

### Advanced
7. Design a system that needs different consistency levels for different operations.
8. How would you handle a network partition in a distributed database?
9. Explain how quorum-based systems make CAP trade-offs.

---

[← Previous: Consistency](../03-consistency/README.md) | [Next: Latency vs Throughput →](../05-latency-throughput/README.md)
