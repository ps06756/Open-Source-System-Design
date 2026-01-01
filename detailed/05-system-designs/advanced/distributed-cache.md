# Design a Distributed Cache

## 1. Requirements

### Functional Requirements
- Get/Set/Delete operations
- TTL (Time-To-Live) support
- Support various data types
- Eviction policies (LRU, LFU)

### Non-Functional Requirements
- Sub-millisecond latency
- High availability
- Horizontal scalability
- Consistency (configurable)
- Handle hot keys

### Scale Estimates
- 1M requests per second
- 1 TB total cache size
- 1 KB average value size
- 99.9% cache hit rate target

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Client                               │ │
│  │  (Application with cache client library)               │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│                           ▼                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │               Consistent Hashing                        │ │
│  │         (Route key to correct shard)                   │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│         ┌─────────────────┼─────────────────┐               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │   Cache     │   │   Cache     │   │   Cache     │       │
│  │   Node 1    │   │   Node 2    │   │   Node 3    │       │
│  │   (Shard 1) │   │   (Shard 2) │   │   (Shard 3) │       │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘       │
│         │                 │                 │               │
│         ▼                 ▼                 ▼               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │  Replica    │   │  Replica    │   │  Replica    │       │
│  │  Node 1'    │   │  Node 2'    │   │  Node 3'    │       │
│  └─────────────┘   └─────────────┘   └─────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Data Partitioning (Consistent Hashing)

```
┌─────────────────────────────────────────────────────────────┐
│                  Consistent Hashing                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Why not simple modulo hashing?                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  hash(key) % N = shard                                 │ │
│  │                                                         │ │
│  │  Problem: Add/remove node → ALL keys rehash!           │ │
│  │  N=3: key%3=1 → Node 1                                 │ │
│  │  N=4: key%4=2 → Node 2 (cache miss!)                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Consistent Hashing:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │            ●─────── Node A                              │ │
│  │         ╱                 ╲                             │ │
│  │       ╱                     ╲                           │ │
│  │     ●                         ●───── key1 → Node B     │ │
│  │   Node D                    Node B                      │ │
│  │     ╲                       ╱                           │ │
│  │       ╲                   ╱                             │ │
│  │         ╲               ╱                               │ │
│  │           ●───────────●                                 │ │
│  │         Node C       key2                               │ │
│  │                      → Node C                           │ │
│  │                                                         │ │
│  │  Keys assigned to next node clockwise                  │ │
│  │                                                         │ │
│  │  Add/remove node: Only K/N keys rehash                 │ │
│  │  (K = total keys, N = nodes)                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Virtual Nodes (for balance):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Each physical node → 100-200 virtual nodes           │ │
│  │  Distributes keys more evenly                         │ │
│  │  Helps when nodes have different capacity             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Replication

```
┌─────────────────────────────────────────────────────────────┐
│                     Replication                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Primary-Replica Model:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Write path:                                            │ │
│  │  Client ──▶ Primary ──▶ Replica 1                      │ │
│  │                     └──▶ Replica 2                      │ │
│  │                                                         │ │
│  │  Read path (configurable):                             │ │
│  │  Option A: Read from Primary (strong consistency)      │ │
│  │  Option B: Read from any (eventual consistency)        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Write Consistency Options:                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  W=1: Return after primary ACK                         │ │
│  │       ✓ Fast writes                                    │ │
│  │       ✗ Data loss if primary fails                    │ │
│  │                                                         │ │
│  │  W=majority: Return after majority ACK                 │ │
│  │       ✓ Durable                                        │ │
│  │       ✗ Higher latency                                │ │
│  │                                                         │ │
│  │  W=all: Return after all replicas ACK                 │ │
│  │       ✓ Strongest durability                          │ │
│  │       ✗ Slowest, blocked by slowest replica           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Replication Strategy:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Synchronous:                                           │ │
│  │  Write ──▶ Primary ──sync──▶ Replica ──▶ ACK          │ │
│  │  ✓ Strong consistency                                  │ │
│  │  ✗ Higher latency                                      │ │
│  │                                                         │ │
│  │  Asynchronous:                                          │ │
│  │  Write ──▶ Primary ──ACK──▶ Client                    │ │
│  │                    └──async──▶ Replica                 │ │
│  │  ✓ Lower latency                                       │ │
│  │  ✗ Potential data loss                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Eviction Policies

```
┌─────────────────────────────────────────────────────────────┐
│                  Eviction Policies                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  When cache is full, which keys to evict?                  │
│                                                              │
│  LRU (Least Recently Used):                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Data structure: HashMap + Doubly Linked List          │ │
│  │                                                         │ │
│  │  ┌────────────────────────────────────────────────┐   │ │
│  │  │  Head                               Tail       │   │ │
│  │  │  (Most recent)                    (Least)      │   │ │
│  │  │  ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐      │   │ │
│  │  │  │ A │◀─▶│ B │◀─▶│ C │◀─▶│ D │◀─▶│ E │      │   │ │
│  │  │  └───┘   └───┘   └───┘   └───┘   └───┘      │   │ │
│  │  │           ↑                         ↑        │   │ │
│  │  │         access                    evict      │   │ │
│  │  └────────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  On access: Move to head (O(1))                        │ │
│  │  On evict: Remove from tail (O(1))                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  LFU (Least Frequently Used):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Track access count per key                            │ │
│  │  Evict key with lowest count                          │ │
│  │                                                         │ │
│  │  ✓ Better for stable access patterns                  │ │
│  │  ✗ New items can be evicted too quickly               │ │
│  │                                                         │ │
│  │  Solution: LFU with aging (decay counts over time)    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  TTL (Time-To-Live):                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Each key has expiration time                         │ │
│  │                                                         │ │
│  │  Implementation:                                        │ │
│  │  • Lazy expiration: Check on access                   │ │
│  │  • Active expiration: Background thread scans keys    │ │
│  │  • Hybrid: Both (Redis approach)                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Hot Key Problem

```
┌─────────────────────────────────────────────────────────────┐
│                   Hot Key Problem                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Some keys get massive traffic                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Example: "Taylor Swift concert tickets"               │ │
│  │  Single key → Single node → Overloaded!               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Solutions:                                                  │
│                                                              │
│  1. Local Cache (L1 Cache):                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Application ──▶ Local Cache (in-memory)               │ │
│  │                        ↓ miss                          │ │
│  │                  Distributed Cache                     │ │
│  │                                                         │ │
│  │  Hot keys cached locally, reduce distributed calls    │ │
│  │  TTL: Very short (seconds) to avoid staleness         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Key Replication:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Replicate hot key to multiple shards                 │ │
│  │                                                         │ │
│  │  hot_key:0 → Shard 1                                  │ │
│  │  hot_key:1 → Shard 2                                  │ │
│  │  hot_key:2 → Shard 3                                  │ │
│  │                                                         │ │
│  │  Client randomly appends 0-N to spread load           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Consistent Hashing with Load Awareness:                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  If target node is overloaded, try next node         │ │
│  │  Requires coordination/monitoring                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Cache Coherence

```
┌─────────────────────────────────────────────────────────────┐
│                   Cache Coherence                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Cache and database can diverge                   │
│                                                              │
│  Cache-Aside Pattern:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Read:                                                  │ │
│  │  1. Check cache                                        │ │
│  │  2. If miss, read from DB                             │ │
│  │  3. Write to cache                                    │ │
│  │                                                         │ │
│  │  Write:                                                 │ │
│  │  1. Write to DB                                        │ │
│  │  2. Invalidate cache (NOT update!)                    │ │
│  │                                                         │ │
│  │  Why invalidate vs update?                             │ │
│  │  Race condition:                                        │ │
│  │  Thread 1: Write A=1 to DB                            │ │
│  │  Thread 2: Write A=2 to DB, update cache A=2          │ │
│  │  Thread 1: Update cache A=1                           │ │
│  │  Result: DB=2, Cache=1 (inconsistent!)                │ │
│  │                                                         │ │
│  │  Invalidate avoids this race condition                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Write-Through vs Write-Behind:                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Write-Through:                                         │ │
│  │  App ──▶ Cache ──▶ DB (sync)                          │ │
│  │  ✓ Consistent                                          │ │
│  │  ✗ Higher write latency                               │ │
│  │                                                         │ │
│  │  Write-Behind:                                          │ │
│  │  App ──▶ Cache ──(async)──▶ DB                        │ │
│  │  ✓ Fast writes                                         │ │
│  │  ✗ Potential data loss                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Partitioning** | Consistent hashing | Minimal rehashing |
| **Replication** | Primary-replica | HA + read scaling |
| **Eviction** | LRU with TTL | Good general purpose |
| **Hot Keys** | Local cache + replication | Spread load |
| **Consistency** | Cache-aside with invalidate | Avoid race conditions |

---

[Back to System Designs](../README.md)
