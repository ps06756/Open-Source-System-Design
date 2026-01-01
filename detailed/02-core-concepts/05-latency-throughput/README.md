# Latency vs Throughput

Understanding the relationship between latency and throughput is crucial for designing performant systems and making the right trade-offs.

## Table of Contents
1. [Definitions](#definitions)
2. [The Relationship](#the-relationship)
3. [Measuring Performance](#measuring-performance)
4. [Optimization Strategies](#optimization-strategies)
5. [Interview Questions](#interview-questions)

---

## Definitions

```
┌─────────────────────────────────────────────────────────────┐
│                 Latency vs Throughput                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LATENCY                                                     │
│  Time to complete a single request                          │
│                                                              │
│  Request ─────────────────────────────────▶ Response        │
│           │◀──────── 200ms latency ────────▶│               │
│                                                              │
│  THROUGHPUT                                                  │
│  Number of requests completed per unit time                 │
│                                                              │
│  ─────────────────── 1 second ───────────────────           │
│  │ req1 │ req2 │ req3 │ req4 │ req5 │ ... │ req100 │       │
│  └──────────────────────────────────────────────────┘       │
│  Throughput: 100 requests/second                            │
│                                                              │
│  BANDWIDTH                                                   │
│  Maximum data transfer rate                                  │
│  "Width of the pipe"                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Latency Numbers

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        Latency Reference Numbers                            │
├──────────────────────────────────────────┬─────────────────────────────────┤
│ Operation                                │ Latency                          │
├──────────────────────────────────────────┼─────────────────────────────────┤
│ L1 cache reference                       │ 0.5 ns                          │
│ L2 cache reference                       │ 7 ns                            │
│ Main memory reference                    │ 100 ns                          │
│ SSD random read                          │ 150 μs                          │
│ Read 1MB from memory                     │ 250 μs                          │
│ Round trip in same datacenter            │ 500 μs                          │
│ Read 1MB from SSD                        │ 1 ms                            │
│ Disk seek                                │ 10 ms                           │
│ Read 1MB from disk                       │ 20 ms                           │
│ Send packet CA → Netherlands → CA        │ 150 ms                          │
├──────────────────────────────────────────┼─────────────────────────────────┤
│ Takeaways:                                                                  │
│ • Memory is ~100x faster than SSD                                          │
│ • SSD is ~100x faster than disk                                            │
│ • Network latency dominates in distributed systems                         │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## The Relationship

### Not Inverse!

```
┌─────────────────────────────────────────────────────────────┐
│           Latency and Throughput Relationship                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Common misconception:                                       │
│  "Lower latency = Higher throughput"  ← NOT ALWAYS TRUE     │
│                                                              │
│  Reality:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  High latency can have high throughput                 │ │
│  │  (Example: cargo ship vs sports car)                   │ │
│  │                                                         │ │
│  │  Cargo ship: 3 weeks latency, 100,000 containers      │ │
│  │  Sports car: 3 days latency, 10 boxes                  │ │
│  │                                                         │ │
│  │  Ship throughput >> Car throughput                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  In software:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Batch processing: High latency, high throughput       │ │
│  │  Real-time API: Low latency, variable throughput       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Little's Law

```
┌─────────────────────────────────────────────────────────────┐
│                      Little's Law                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│     L = λ × W                                               │
│                                                              │
│  L = Average number of items in system (concurrency)        │
│  λ = Throughput (arrival rate)                              │
│  W = Average latency (wait time)                            │
│                                                              │
│  Example:                                                    │
│  Server handles 100 req/sec, each takes 50ms                │
│                                                              │
│  L = 100 × 0.05 = 5 concurrent requests                    │
│                                                              │
│  Implications:                                               │
│  • To handle 1000 req/sec at 50ms: need 50 concurrent slots│
│  • To reduce latency to 10ms: same throughput needs only 10│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Measuring Performance

### Percentiles

```
┌─────────────────────────────────────────────────────────────┐
│                      Percentiles                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Why percentiles, not averages?                             │
│                                                              │
│  Average can hide problems:                                  │
│  99 requests at 50ms + 1 request at 5000ms                  │
│  Average = 100ms  ← Looks okay!                             │
│  p99 = 5000ms     ← Shows the problem!                      │
│                                                              │
│  Common percentiles:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ p50 (median):  50% of requests faster than this        │ │
│  │ p90:          90% of requests faster than this        │ │
│  │ p95:          95% of requests faster than this        │ │
│  │ p99:          99% of requests faster than this        │ │
│  │ p99.9:        99.9% of requests faster                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Visualized:                                                 │
│                                                              │
│  Requests sorted by latency:                                │
│  [fast ──────────────────────────────────────────── slow]   │
│        │         │         │                   │             │
│       p50       p90       p95                 p99            │
│       50ms      100ms     150ms              500ms           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Tail Latency Amplification

```
┌─────────────────────────────────────────────────────────────┐
│              Tail Latency Amplification                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  One request may fan out to many backend services:          │
│                                                              │
│       ┌─────────┐                                           │
│       │ Request │                                           │
│       └────┬────┘                                           │
│            │                                                 │
│    ┌───────┼───────┬───────┬───────┬───────┐               │
│    ▼       ▼       ▼       ▼       ▼       ▼               │
│  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐                 │
│  │ A │  │ B │  │ C │  │ D │  │ E │  │ F │                 │
│  │5ms│  │5ms│  │5ms│  │5ms│  │5ms│  │500ms│               │
│  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘                 │
│                                          ▲                  │
│                               Slowest one wins!             │
│                                                              │
│  If each service has 1% chance of being slow:              │
│                                                              │
│  Fan-out to N services:                                     │
│  P(at least one slow) = 1 - 0.99^N                         │
│                                                              │
│  N=1:   1% requests slow                                    │
│  N=10:  10% requests slow                                   │
│  N=50:  40% requests slow                                   │
│  N=100: 63% requests slow                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Optimization Strategies

### Latency Optimization

```
┌─────────────────────────────────────────────────────────────┐
│              Latency Optimization Strategies                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Caching                                                  │
│     Cache hot data in memory                                │
│     100ns (cache) vs 10ms (database)                       │
│                                                              │
│  2. Edge Computing / CDN                                    │
│     Serve from nearest location                             │
│     Reduce network round trips                              │
│                                                              │
│  3. Connection Pooling                                       │
│     Reuse connections, avoid handshake                      │
│     Save 1-3 round trips per request                        │
│                                                              │
│  4. Async Processing                                         │
│     Return immediately, process in background               │
│     User doesn't wait for slow operations                   │
│                                                              │
│  5. Data Locality                                            │
│     Keep related data together                              │
│     Reduce cross-datacenter calls                           │
│                                                              │
│  6. Reduce Payload Size                                      │
│     Compression, selective fields                           │
│     Less data = faster transfer                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Throughput Optimization

```
┌─────────────────────────────────────────────────────────────┐
│             Throughput Optimization Strategies               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Horizontal Scaling                                       │
│     Add more servers                                        │
│     Linear increase in capacity                             │
│                                                              │
│  2. Batching                                                 │
│     Group operations together                               │
│     Amortize overhead across multiple items                 │
│                                                              │
│  3. Parallel Processing                                      │
│     Process multiple requests simultaneously                │
│     Use all available CPU cores                             │
│                                                              │
│  4. Non-blocking I/O                                         │
│     Don't wait for I/O operations                          │
│     Handle more concurrent connections                      │
│                                                              │
│  5. Load Balancing                                           │
│     Distribute work evenly                                  │
│     Avoid hot spots                                         │
│                                                              │
│  6. Queue Buffering                                          │
│     Absorb traffic spikes                                   │
│     Smooth out load                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between latency and throughput?
2. Why do we use percentiles instead of averages?
3. What is tail latency?

### Intermediate
4. Explain Little's Law. How is it useful?
5. What is tail latency amplification and how do you mitigate it?
6. How would you optimize a system for low latency vs high throughput?

### Advanced
7. Design a system that needs both low latency and high throughput.
8. How do you handle latency spikes in a distributed system?
9. Explain the trade-offs between batching and real-time processing.

---

[← Previous: CAP Theorem](../04-cap-theorem/README.md) | [Next: Data Partitioning →](../06-partitioning/README.md)
