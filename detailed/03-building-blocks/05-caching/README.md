# Caching

Caching stores frequently accessed data in fast storage to reduce latency and database load. It's one of the most effective performance optimization techniques.

## Table of Contents
1. [Cache Hierarchy](#cache-hierarchy)
2. [Caching Strategies](#caching-strategies)
3. [Cache Eviction Policies](#cache-eviction-policies)
4. [Redis vs Memcached](#redis-vs-memcached)
5. [Interview Questions](#interview-questions)

---

## Cache Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    Cache Hierarchy                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Speed     Layer              Latency      Size             │
│  ─────     ─────              ───────      ────             │
│                                                              │
│  Fastest   CPU L1 Cache       ~1 ns        32 KB            │
│    │       CPU L2 Cache       ~4 ns        256 KB           │
│    │       CPU L3 Cache       ~15 ns       8 MB             │
│    │       RAM                ~100 ns      16-256 GB        │
│    │       Application Cache  ~100 ns      GB               │
│    │       Distributed Cache  ~1 ms        TB               │
│    │       SSD                ~100 µs      TB               │
│    ▼       HDD                ~10 ms       TB               │
│  Slowest   Network (DB)       ~1-100 ms    PB               │
│                                                              │
│  Rule: Cache hot data at the fastest affordable layer       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Caching Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                  Caching Strategies                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Cache-Aside (Lazy Loading)                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Read:                                                  │ │
│  │  1. Check cache                                        │ │
│  │  2. If miss → read DB → update cache                  │ │
│  │  3. Return data                                        │ │
│  │                                                         │ │
│  │  Write:                                                 │ │
│  │  1. Write to DB                                        │ │
│  │  2. Invalidate cache                                   │ │
│  │                                                         │ │
│  │  ✓ Only caches what's needed                          │ │
│  │  ✗ Cache miss = slow first request                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Write-Through                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Write:                                                 │ │
│  │  1. Write to cache AND DB together                     │ │
│  │  2. Return success                                     │ │
│  │                                                         │ │
│  │  ✓ Cache always consistent                             │ │
│  │  ✗ Higher write latency                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Write-Behind (Write-Back)                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Write:                                                 │ │
│  │  1. Write to cache only                                │ │
│  │  2. Async write to DB later                           │ │
│  │                                                         │ │
│  │  ✓ Fast writes                                         │ │
│  │  ✗ Risk of data loss if cache fails                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  4. Read-Through                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Read:                                                  │ │
│  │  Cache handles DB read on miss                        │ │
│  │  (Cache library manages DB connection)                 │ │
│  │                                                         │ │
│  │  ✓ Simplified application code                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Cache Eviction Policies

```
┌─────────────────────────────────────────────────────────────┐
│                 Eviction Policies                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  When cache is full, which item to remove?                  │
│                                                              │
│  1. LRU (Least Recently Used)                               │
│     Remove item that hasn't been accessed longest           │
│     Most common, good general-purpose policy                │
│                                                              │
│  2. LFU (Least Frequently Used)                             │
│     Remove item with lowest access count                    │
│     Good for stable access patterns                         │
│                                                              │
│  3. FIFO (First In, First Out)                              │
│     Remove oldest item                                       │
│     Simple but not access-aware                             │
│                                                              │
│  4. TTL (Time To Live)                                       │
│     Remove items after expiration time                      │
│     Good for time-sensitive data                            │
│                                                              │
│  5. Random                                                   │
│     Remove random item                                       │
│     Simple, surprisingly effective                          │
│                                                              │
│  Best practice: LRU + TTL combined                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Redis vs Memcached

```
┌────────────────────────────────────────────────────────────────────────────┐
│                       Redis vs Memcached                                    │
├────────────────────────────────────┬───────────────────────────────────────┤
│              Redis                 │            Memcached                   │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Data structures (lists, sets,     │ Simple key-value only                  │
│ sorted sets, hashes)              │                                         │
│                                    │                                         │
│ Persistence options               │ No persistence                          │
│                                    │                                         │
│ Single-threaded                   │ Multi-threaded                          │
│                                    │                                         │
│ Pub/Sub, Lua scripting           │ Simpler feature set                     │
│                                    │                                         │
│ Clustering built-in              │ Client-side sharding                    │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Use for:                          │ Use for:                                │
│ • Sessions                        │ • Simple caching                        │
│ • Leaderboards                    │ • HTML fragments                        │
│ • Rate limiting                   │ • Large memory needs                    │
│ • Real-time analytics             │ • Multi-threaded performance            │
└────────────────────────────────────┴───────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is caching and why is it important?
2. Explain cache-aside pattern.

### Intermediate
3. How do you handle cache invalidation?
4. When would you use Redis vs Memcached?

### Advanced
5. Design a distributed caching system for a global application.

---

[← Previous: Databases](../04-databases/README.md) | [Next: Message Queues →](../06-message-queues/README.md)
