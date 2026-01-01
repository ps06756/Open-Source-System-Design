# Scalability

Scalability is the capability of a system to handle a growing amount of work by adding resources. Understanding scaling strategies is fundamental to system design.

## Table of Contents
1. [What is Scalability?](#what-is-scalability)
2. [Vertical Scaling](#vertical-scaling)
3. [Horizontal Scaling](#horizontal-scaling)
4. [Scaling Strategies](#scaling-strategies)
5. [Scaling Databases](#scaling-databases)
6. [Interview Questions](#interview-questions)

---

## What is Scalability?

```
┌─────────────────────────────────────────────────────────────┐
│                   Scalability Dimensions                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Load Scalability                                         │
│     Handle more requests/users/data                          │
│                                                              │
│  2. Geographic Scalability                                   │
│     Serve users across regions with low latency              │
│                                                              │
│  3. Administrative Scalability                               │
│     Multiple organizations can share the system              │
│                                                              │
│  Metrics to measure:                                         │
│  • Throughput (requests/second)                             │
│  • Response time (latency)                                   │
│  • Concurrent users                                          │
│  • Data volume                                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Vertical Scaling

### Scale Up: Add More Power

```
┌─────────────────────────────────────────────────────────────┐
│                   Vertical Scaling                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Before:                      After:                         │
│  ┌──────────────┐            ┌──────────────┐               │
│  │   Server     │            │   Server     │               │
│  │              │            │              │               │
│  │  4 CPU       │    ───▶    │  32 CPU      │               │
│  │  16GB RAM    │            │  256GB RAM   │               │
│  │  500GB SSD   │            │  4TB NVMe    │               │
│  └──────────────┘            └──────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Simple to implement | Hardware limits |
| No code changes needed | Single point of failure |
| No data distribution complexity | Expensive at high end |
| Lower latency (no network hops) | Downtime during upgrade |

### When to Use

- **Database servers** where data consistency is critical
- **Legacy applications** that can't be easily distributed
- **Quick wins** before investing in horizontal scaling
- **Stateful applications** with complex state management

---

## Horizontal Scaling

### Scale Out: Add More Machines

```
┌─────────────────────────────────────────────────────────────┐
│                  Horizontal Scaling                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Before:                                                     │
│  ┌──────────────┐                                           │
│  │   Server     │                                           │
│  │   (1x)       │                                           │
│  └──────────────┘                                           │
│                                                              │
│  After:                                                      │
│                    ┌─────────────────┐                      │
│                    │  Load Balancer  │                      │
│                    └────────┬────────┘                      │
│           ┌─────────────────┼─────────────────┐             │
│           │                 │                 │             │
│           ▼                 ▼                 ▼             │
│     ┌──────────┐     ┌──────────┐     ┌──────────┐         │
│     │ Server 1 │     │ Server 2 │     │ Server N │         │
│     └──────────┘     └──────────┘     └──────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Theoretically unlimited scale | Increased complexity |
| Better fault tolerance | Data consistency challenges |
| Cost-effective (commodity hardware) | Network latency |
| No single point of failure | Operational overhead |

### Requirements for Horizontal Scaling

```
┌─────────────────────────────────────────────────────────────┐
│           Requirements for Horizontal Scaling                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Stateless Application Servers                            │
│     ┌──────────────────────────────────────────────────┐    │
│     │ ✗ Bad: Session stored in server memory           │    │
│     │ ✓ Good: Session in Redis/database               │    │
│     └──────────────────────────────────────────────────┘    │
│                                                              │
│  2. Shared Storage                                           │
│     ┌──────────────────────────────────────────────────┐    │
│     │ ✗ Bad: Files on local disk                       │    │
│     │ ✓ Good: S3, NFS, or distributed file system      │    │
│     └──────────────────────────────────────────────────┘    │
│                                                              │
│  3. Database Scaling Strategy                                │
│     ┌──────────────────────────────────────────────────┐    │
│     │ - Read replicas for read-heavy workloads        │    │
│     │ - Sharding for write-heavy workloads            │    │
│     │ - Caching to reduce database load               │    │
│     └──────────────────────────────────────────────────┘    │
│                                                              │
│  4. Load Balancing                                           │
│     ┌──────────────────────────────────────────────────┐    │
│     │ - Distribute traffic across servers             │    │
│     │ - Health checks for failover                    │    │
│     │ - Session affinity if needed                    │    │
│     └──────────────────────────────────────────────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Scaling Strategies

### 1. Functional Decomposition (Microservices)

```
┌─────────────────────────────────────────────────────────────┐
│              Functional Decomposition                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Monolith:                                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Single Application                   │    │
│  │   [Users] [Products] [Orders] [Payments] [Search]   │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  Microservices:                                              │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │  User   │ │ Product │ │  Order  │ │ Payment │           │
│  │ Service │ │ Service │ │ Service │ │ Service │           │
│  │ (3 inst)│ │ (5 inst)│ │ (10 inst│ │ (2 inst)│           │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                              │
│  Benefits:                                                   │
│  • Scale services independently                             │
│  • Different tech stacks per service                        │
│  • Smaller, focused teams                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. Data Partitioning (Sharding)

```
┌─────────────────────────────────────────────────────────────┐
│                   Data Partitioning                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Before: Single database                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              All User Data (100M users)              │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  After: Sharded by user_id                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │  Shard 1    │ │  Shard 2    │ │  Shard 3    │           │
│  │ Users A-H   │ │ Users I-P   │ │ Users Q-Z   │           │
│  │ (33M users) │ │ (33M users) │ │ (33M users) │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
│                                                              │
│  Sharding Strategies:                                        │
│  • Range-based (A-H, I-P, Q-Z)                              │
│  • Hash-based (hash(user_id) % num_shards)                  │
│  • Geographic (US, EU, Asia)                                │
│  • Directory-based (lookup table)                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3. Caching

```
┌─────────────────────────────────────────────────────────────┐
│                    Caching Layers                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐                                                │
│  │ Browser │──── Browser Cache (static assets)              │
│  └────┬────┘                                                │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────┐                                                │
│  │   CDN   │──── Edge Cache (global distribution)           │
│  └────┬────┘                                                │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────┐                                                │
│  │   App   │──── Application Cache (computed data)          │
│  └────┬────┘                                                │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────┐                                                │
│  │  Redis  │──── Distributed Cache (shared state)           │
│  └────┬────┘                                                │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────┐                                                │
│  │Database │──── Query Cache (frequent queries)             │
│  └─────────┘                                                │
│                                                              │
│  Rule of thumb: Cache data that is                          │
│  • Read frequently                                           │
│  • Expensive to compute                                      │
│  • Rarely changes                                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4. Asynchronous Processing

```
┌─────────────────────────────────────────────────────────────┐
│              Asynchronous Processing                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Synchronous (Blocking):                                     │
│  User ──▶ API ──▶ Process ──▶ DB ──▶ Email ──▶ Response    │
│              Total time: 500ms + 100ms + 2000ms = 2600ms    │
│                                                              │
│  Asynchronous (Non-blocking):                                │
│  User ──▶ API ──▶ Queue ──▶ Response (200ms)               │
│                    │                                         │
│                    └──▶ Worker ──▶ Email (background)       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                                                       │   │
│  │  ┌─────────┐        ┌─────────┐        ┌─────────┐  │   │
│  │  │   API   │───────▶│  Queue  │───────▶│ Workers │  │   │
│  │  │ Servers │        │ (Kafka) │        │         │  │   │
│  │  └─────────┘        └─────────┘        └─────────┘  │   │
│  │       │                                     │        │   │
│  │       │ Quick response                      │        │   │
│  │       ▼                                     ▼        │   │
│  │    [User]                              [Process]     │   │
│  │                                                       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Use cases:                                                  │
│  • Email/SMS notifications                                  │
│  • Image/video processing                                   │
│  • Report generation                                        │
│  • Data aggregation                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Scaling Databases

### Read Scaling with Replicas

```
┌─────────────────────────────────────────────────────────────┐
│                   Read Replicas                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    ┌───────────────┐                        │
│                    │    Primary    │                        │
│                    │    (Write)    │                        │
│                    └───────┬───────┘                        │
│                            │ Replication                    │
│           ┌────────────────┼────────────────┐               │
│           │                │                │               │
│           ▼                ▼                ▼               │
│     ┌──────────┐     ┌──────────┐     ┌──────────┐         │
│     │ Replica 1│     │ Replica 2│     │ Replica 3│         │
│     │  (Read)  │     │  (Read)  │     │  (Read)  │         │
│     └──────────┘     └──────────┘     └──────────┘         │
│                                                              │
│  Write path: Client → Primary                               │
│  Read path:  Client → Any Replica                           │
│                                                              │
│  Trade-off: Eventual consistency on reads                   │
│  (Replica might be slightly behind primary)                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Write Scaling with Sharding

```
┌─────────────────────────────────────────────────────────────┐
│                    Sharding Strategies                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Range-based Sharding                                     │
│     user_id 1-1M    → Shard 1                               │
│     user_id 1M-2M   → Shard 2                               │
│     ⚠️ Risk: Hotspots if range is uneven                    │
│                                                              │
│  2. Hash-based Sharding                                      │
│     shard = hash(user_id) % num_shards                      │
│     ✓ Even distribution                                      │
│     ⚠️ Harder to add shards (rehashing)                     │
│                                                              │
│  3. Consistent Hashing                                       │
│     ┌─────────────────────────────────────────────┐         │
│     │           Ring (0 to 2^32-1)                │         │
│     │                                              │         │
│     │         Node A      Node B                  │         │
│     │           ●───────────●                     │         │
│     │          ╱             ╲                    │         │
│     │         ╱               ╲                   │         │
│     │        ●─────────────────●                  │         │
│     │     Node D            Node C                │         │
│     │                                              │         │
│     │  Keys are assigned to next node clockwise   │         │
│     │  Adding/removing node affects minimal keys  │         │
│     └─────────────────────────────────────────────┘         │
│                                                              │
│  4. Directory-based Sharding                                 │
│     Lookup service maps key → shard                         │
│     ✓ Flexible                                               │
│     ⚠️ Lookup service is single point of failure            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between vertical and horizontal scaling?
2. What are the advantages of horizontal scaling?
3. Why do applications need to be stateless for horizontal scaling?

### Intermediate
4. Describe different database sharding strategies. When would you use each?
5. How would you scale a read-heavy vs write-heavy application?
6. What is consistent hashing and why is it useful?

### Advanced
7. Design a system that can handle 10x traffic growth in a month.
8. How would you migrate from a monolith to microservices?
9. What are the challenges of cross-shard queries and how do you handle them?

---

[← Back to Module](../README.md) | [Next: Availability →](../02-availability/README.md)
