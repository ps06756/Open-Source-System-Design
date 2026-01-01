# Data Partitioning

Partitioning (also called sharding) is essential for scaling databases beyond a single machine. It distributes data across multiple nodes.

## Table of Contents
1. [Why Partition?](#why-partition)
2. [Partitioning Strategies](#partitioning-strategies)
3. [Consistent Hashing](#consistent-hashing)
4. [Challenges](#challenges)
5. [Interview Questions](#interview-questions)

---

## Why Partition?

```
┌─────────────────────────────────────────────────────────────┐
│                  Why Data Partitioning?                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Single database bottleneck:                                 │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 100M users                           │    │
│  │                 1TB data                             │    │
│  │                 10K writes/sec                       │    │
│  │                                                      │    │
│  │            ┌─────────────────────┐                  │    │
│  │            │   Single Database   │                  │    │
│  │            │   (Can't handle!)   │                  │    │
│  │            └─────────────────────┘                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 Sharded:                             │    │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐            │    │
│  │   │ Shard 1 │  │ Shard 2 │  │ Shard N │            │    │
│  │   │  33M    │  │  33M    │  │  33M    │            │    │
│  │   │ 333GB   │  │ 333GB   │  │ 333GB   │            │    │
│  │   │ 3.3K/s  │  │ 3.3K/s  │  │ 3.3K/s  │            │    │
│  │   └─────────┘  └─────────┘  └─────────┘            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Benefits:                                                   │
│  • Horizontal write scaling                                 │
│  • More storage capacity                                     │
│  • Better locality (geographic sharding)                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Partitioning Strategies

### Range-Based Partitioning

```
┌─────────────────────────────────────────────────────────────┐
│                 Range-Based Partitioning                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Partition by ranges of the key:                            │
│                                                              │
│  By user_id:                                                 │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │  Shard 1    │  Shard 2    │  Shard 3    │               │
│  │  1-1M       │  1M-2M      │  2M-3M      │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                              │
│  By timestamp:                                               │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │  Jan 2024   │  Feb 2024   │  Mar 2024   │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                              │
│  Pros:                                                       │
│  ✓ Range queries efficient                                  │
│  ✓ Simple to understand                                     │
│  ✓ Good for time-series data                               │
│                                                              │
│  Cons:                                                       │
│  ✗ Hotspots (new users → last shard)                       │
│  ✗ Uneven distribution if ranges unequal                   │
│  ✗ Rebalancing can be complex                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Hash-Based Partitioning

```
┌─────────────────────────────────────────────────────────────┐
│                  Hash-Based Partitioning                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  shard_id = hash(key) % num_shards                          │
│                                                              │
│  user_id = 12345                                            │
│  hash(12345) = 789456                                       │
│  shard = 789456 % 3 = 0  → Shard 0                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                                                       │  │
│  │     hash("alice") % 3 = 0  ─────▶  ┌─────────────┐   │  │
│  │     hash("bob") % 3 = 1    ─────▶  │  Shard 0    │   │  │
│  │     hash("carol") % 3 = 2  ─────▶  │  Shard 1    │   │  │
│  │     hash("dave") % 3 = 0   ─────▶  │  Shard 2    │   │  │
│  │     hash("eve") % 3 = 1    ─────▶  └─────────────┘   │  │
│  │                                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Pros:                                                       │
│  ✓ Even distribution of data                               │
│  ✓ No hotspots (if hash is good)                           │
│                                                              │
│  Cons:                                                       │
│  ✗ Range queries require hitting all shards                │
│  ✗ Adding shards = massive rehashing                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Directory-Based Partitioning

```
┌─────────────────────────────────────────────────────────────┐
│               Directory-Based Partitioning                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Lookup service maps keys to shards:                        │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Lookup Service                     │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │  user_id 1-50K      → Shard A                │   │   │
│  │  │  user_id 50K-200K   → Shard B                │   │   │
│  │  │  user_id 200K-500K  → Shard C                │   │   │
│  │  │  user_id 500K+      → Shard D                │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│              │                                              │
│              ▼                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ Shard A │  │ Shard B │  │ Shard C │  │ Shard D │       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                              │
│  Pros:                                                       │
│  ✓ Flexible placement                                       │
│  ✓ Easy to move data                                        │
│  ✓ Can handle uneven distribution                          │
│                                                              │
│  Cons:                                                       │
│  ✗ Lookup service is SPOF                                  │
│  ✗ Extra network hop                                        │
│  ✗ Lookup service must scale                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Consistent Hashing

### The Problem with Modulo Hashing

```
┌─────────────────────────────────────────────────────────────┐
│             Problem: Adding/Removing Shards                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Before: 3 shards                                           │
│  hash(key) % 3 = shard                                      │
│                                                              │
│  key A: hash = 10 → 10 % 3 = 1 → Shard 1                   │
│  key B: hash = 20 → 20 % 3 = 2 → Shard 2                   │
│  key C: hash = 30 → 30 % 3 = 0 → Shard 0                   │
│                                                              │
│  After: 4 shards                                            │
│  hash(key) % 4 = shard                                      │
│                                                              │
│  key A: hash = 10 → 10 % 4 = 2 → Shard 2 ← MOVED!         │
│  key B: hash = 20 → 20 % 4 = 0 → Shard 0 ← MOVED!         │
│  key C: hash = 30 → 30 % 4 = 2 → Shard 2 ← MOVED!         │
│                                                              │
│  Almost ALL keys move when adding one shard!                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Consistent Hashing Solution

```
┌─────────────────────────────────────────────────────────────┐
│                   Consistent Hashing                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Imagine a ring from 0 to 2^32-1:                          │
│                                                              │
│                    0                                        │
│                    ●                                        │
│              Node A │ Node B                               │
│                ●────┼────●                                 │
│               ╱      │      ╲                              │
│              ╱       │       ╲                             │
│             ●        │        ●                            │
│          Key X       │      Node C                         │
│                      │                                      │
│             ●        │        ●                            │
│          Key Y      │      Key Z                          │
│              ╲       │       ╱                             │
│               ╲      │      ╱                              │
│                ●────┴────●                                 │
│             Node D                                          │
│                                                              │
│  Rule: Key is assigned to NEXT node clockwise              │
│  Key X → Node A (next clockwise)                           │
│                                                              │
│  When Node B is removed:                                    │
│  Only keys between A and B move to C                       │
│  Other keys stay put!                                       │
│                                                              │
│  On average: Only K/N keys move (K=keys, N=nodes)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Virtual Nodes

```
┌─────────────────────────────────────────────────────────────┐
│                     Virtual Nodes                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Uneven distribution with few physical nodes       │
│                                                              │
│  Solution: Each physical node gets multiple virtual nodes   │
│                                                              │
│                    0                                        │
│                    ●                                        │
│               A1 ──┼── B1                                  │
│                ●   │   ●                                   │
│              B2●   │   ●A2                                 │
│               ╱    │    ╲                                  │
│              ╱     │     ╲                                 │
│             ●      │      ●                                │
│           A3       │      B3                               │
│                    │                                        │
│                                                              │
│  Physical nodes: A, B                                       │
│  Virtual nodes: A1, A2, A3, B1, B2, B3                     │
│                                                              │
│  Benefits:                                                   │
│  ✓ More even distribution                                  │
│  ✓ Heterogeneous nodes (powerful node = more vnodes)       │
│  ✓ Smoother rebalancing                                    │
│                                                              │
│  Used by: Cassandra, DynamoDB, Riak                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Challenges

### Cross-Shard Operations

```
┌─────────────────────────────────────────────────────────────┐
│                 Cross-Shard Challenges                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Cross-Shard Joins                                       │
│     User on Shard A, Orders on Shard B                     │
│     JOIN requires scatter-gather across shards             │
│     Slow and complex!                                       │
│                                                              │
│  2. Cross-Shard Transactions                                │
│     Transfer money: Debit Shard A, Credit Shard B          │
│     Need distributed transactions (2PC/Saga)               │
│                                                              │
│  3. Referential Integrity                                   │
│     Foreign keys across shards don't work                  │
│     Must handle in application                             │
│                                                              │
│  Solutions:                                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ • Co-locate related data on same shard              │   │
│  │   (User + User's Orders on same shard)              │   │
│  │                                                      │   │
│  │ • Denormalize to avoid joins                        │   │
│  │                                                      │   │
│  │ • Use application-level transactions (Saga)         │   │
│  │                                                      │   │
│  │ • Accept eventual consistency for cross-shard ops   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Hotspots

```
┌─────────────────────────────────────────────────────────────┐
│                   Handling Hotspots                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Celebrity tweets, viral posts                     │
│           One key gets millions of requests                 │
│           That shard becomes overloaded                     │
│                                                              │
│  Solutions:                                                  │
│                                                              │
│  1. Add random suffix                                       │
│     key = "celebrity_123" + random(0,10)                   │
│     Spreads across 10 shards                               │
│     Read = scatter-gather all 10                           │
│                                                              │
│  2. Dedicated shard for hot keys                           │
│     Monitor and move hot keys to special shard             │
│     With more resources                                     │
│                                                              │
│  3. Caching layer                                           │
│     Cache hot data in front of database                    │
│     Absorbs most read traffic                              │
│                                                              │
│  4. Rate limiting                                           │
│     Limit requests to hot keys                             │
│     Protect the shard                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is data partitioning and why is it needed?
2. Explain the difference between range and hash partitioning.
3. What is consistent hashing?

### Intermediate
4. How does consistent hashing minimize data movement?
5. What are virtual nodes and why are they useful?
6. How would you handle cross-shard queries?

### Advanced
7. Design a sharding strategy for a social media platform.
8. How would you handle hotspots in a sharded database?
9. How do you rebalance data when adding new shards?

---

[← Previous: Latency vs Throughput](../05-latency-throughput/README.md) | [Next: Replication →](../07-replication/README.md)
