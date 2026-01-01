# Latency vs Throughput

Understanding latency and throughput is essential for designing performant systems and making the right trade-offs.

## Table of Contents
- [Definitions](#definitions)
- [Latency Deep Dive](#latency-deep-dive)
- [Throughput Deep Dive](#throughput-deep-dive)
- [The Relationship](#the-relationship)
- [Optimization Strategies](#optimization-strategies)
- [Key Takeaways](#key-takeaways)

## Definitions

### Latency

The time taken to complete a single operation (delay).

```
Request ──────────────────────────────▶ Response
        ◀────────── Latency ──────────▶
                   (e.g., 50ms)
```

### Throughput

The number of operations completed per unit time (capacity).

```
Time window (1 second)
├──────────────────────────────────────┤
│ ■ ■ ■ ■ ■ ■ ■ ■ ■ ■                 │
│          10 requests                 │
└──────────────────────────────────────┘
Throughput = 10 requests/second
```

### Bandwidth

Maximum data transfer rate of a channel.

```
Bandwidth = 1 Gbps (maximum capacity)
Throughput = 800 Mbps (actual usage)
Latency = 10ms (per packet delay)
```

## Latency Deep Dive

### Types of Latency

| Type | Description | Typical Value |
|------|-------------|---------------|
| Network latency | Time for packet to travel | 0.1-100ms |
| Processing latency | CPU time to process | 1-100ms |
| Queuing latency | Time waiting in queue | Variable |
| Disk latency | Storage access time | 0.01-10ms |

### Latency at Scale

```
Latency Numbers Every Programmer Should Know:

L1 cache reference                           0.5 ns
L2 cache reference                           7   ns
Main memory reference                      100   ns
SSD random read                            150   μs
HDD seek                                    10   ms

Network (same datacenter):
  - Same rack                              0.5   ms
  - Different rack                         1     ms

Network (cross-region):
  - US East → US West                      40    ms
  - US → Europe                            80    ms
  - US → Asia                             150    ms
```

### Percentile Latencies

Average latency hides outliers. Use percentiles:

```
p50 (median):  50% of requests faster than this
p90:           90% of requests faster than this
p99:           99% of requests faster than this
p99.9:         99.9% of requests faster than this

Example distribution:
┌─────────────────────────────────────────────────────┐
│  Latency (ms)                                       │
│                                                     │
│  ████████████████████████████████████       50ms   │ p50
│  ███████████████████████████████████████    80ms   │ p90
│  ██████████████████████████████████████████ 200ms  │ p99
│  ██████████████████████████████████████████ 2000ms │ p99.9
└─────────────────────────────────────────────────────┘
```

### Why p99 Matters

```
1000 requests/second
- p50 = 50ms: Half your users see 50ms or less
- p99 = 200ms: 10 users per second see 200ms+

If each user makes 10 requests:
- 1 in 10 requests is slow
- Most users will experience slowness!
```

### Tail Latency Amplification

In microservices, one slow call affects the whole request:

```
Frontend ─┬─▶ Service A (50ms) ─┐
          │                      │
          ├─▶ Service B (50ms) ─┼─▶ Response
          │                      │
          └─▶ Service C (500ms) ─┘
                  ▲
                  └── One slow service = slow response

Total latency = max(A, B, C) = 500ms (not 50ms)
```

## Throughput Deep Dive

### Measuring Throughput

| Metric | Unit | Context |
|--------|------|---------|
| RPS | Requests/second | API servers |
| QPS | Queries/second | Databases |
| TPS | Transactions/second | Payment systems |
| IOPS | I/O operations/second | Storage |
| Bandwidth | Mbps, Gbps | Network |

### Throughput Limiters

```
┌─────────────────────────────────────────────────────┐
│                 Request Flow                         │
│                                                      │
│  Network ─▶ Load Balancer ─▶ App Server ─▶ Database │
│  10Gbps      100K RPS         5K RPS       1K QPS   │
│                                              ▲       │
│                                              │       │
│                              Bottleneck ─────┘       │
└─────────────────────────────────────────────────────┘

System throughput = min(all components) = 1K QPS
```

### Little's Law

```
L = λ × W

L = Average number of items in system
λ = Average arrival rate (throughput)
W = Average time in system (latency)

Example:
If latency = 100ms and you need 1000 RPS:
L = 1000 × 0.1 = 100 concurrent requests

You need capacity for 100 concurrent connections.
```

### Calculating Throughput Capacity

```
Single server capacity:
- CPU: 4 cores, each handles 100 req/s = 400 RPS
- Memory: 16GB, each request uses 10MB = 1600 concurrent
- Connections: 10K limit

Actual capacity = min(400, 1600, 10000) = 400 RPS

For 10,000 RPS: Need 25 servers (+ buffer)
```

## The Relationship

### Trade-off

Often you must choose between optimizing for latency or throughput:

```
             Low Latency                High Throughput
                 │                            │
                 ▼                            ▼
        ┌──────────────┐            ┌──────────────┐
        │ Process      │            │ Batch        │
        │ immediately  │            │ requests     │
        │              │            │              │
        │ Small queues │            │ Large queues │
        │              │            │              │
        │ More         │            │ Fewer        │
        │ resources    │            │ resources    │
        └──────────────┘            └──────────────┘
```

### Queueing Theory

```
As utilization increases, latency grows exponentially:

Latency
    │                                    ╱
    │                                  ╱
    │                                ╱
    │                              ╱
    │                           ╱
    │                        ╱
    │                    ╱
    │              ╱──
    │        ╱────
    │   ╱────
    └────────────────────────────────────────────▶
    0%          50%        80%   90%  100%
                    Utilization

At 80% utilization, latency starts to spike
At 100%, queue grows infinitely (system failure)
```

### Bandwidth-Delay Product

```
BDP = Bandwidth × Latency

Example:
1 Gbps bandwidth, 50ms latency
BDP = 1,000,000,000 × 0.050 = 50,000,000 bits = 6.25 MB

You need 6.25 MB of data "in flight" to fully utilize the link.
```

## Optimization Strategies

### Reducing Latency

| Strategy | Description |
|----------|-------------|
| Caching | Store results closer to user |
| CDN | Geographic distribution |
| Connection pooling | Reuse connections |
| Async processing | Don't wait for slow operations |
| Denormalization | Reduce joins |
| Indexing | Faster database lookups |
| Compression | Reduce data transfer time |

### Increasing Throughput

| Strategy | Description |
|----------|-------------|
| Horizontal scaling | More servers |
| Load balancing | Distribute requests |
| Batching | Process multiple items together |
| Queuing | Smooth out traffic spikes |
| Caching | Reduce backend load |
| Sharding | Distribute data |
| Connection multiplexing | Reuse connections |

### Batching Trade-off

```
Without batching (low latency, low throughput):
─▶ Process ─▶ Process ─▶ Process ─▶ Process ─▶
   1ms          1ms          1ms          1ms
   4 requests in 4ms = 1000 RPS

With batching (higher latency, higher throughput):
─▶ Wait ─────▶ Process Batch ─▶
   10ms             5ms
   100 requests in 15ms = 6666 RPS
   But each request waits 10ms+ extra
```

### Caching Impact

```
Without cache:
Request ─▶ App ─▶ Database ─▶ Response
             100ms total

With cache (90% hit rate):
Request ─▶ App ─▶ Cache ─▶ Response (90%)
             5ms

Request ─▶ App ─▶ Cache Miss ─▶ DB ─▶ Response (10%)
             105ms

Average: 0.9 × 5 + 0.1 × 105 = 15ms (6.7x improvement)
Throughput: Also 6.7x improvement (fewer DB queries)
```

### Connection Pooling

```
Without pooling:
Each request: Connect (30ms) ─▶ Query (10ms) ─▶ Close
Total: 40ms per request

With pooling:
First request: Connect (30ms) ─▶ Query (10ms)
Next requests: Query (10ms) only (reuse connection)
Average: ~10ms per request (4x improvement)
```

## Key Takeaways

1. **Latency is delay, throughput is capacity** - different optimization strategies
2. **Measure percentiles** - p50, p99, not just average
3. **Little's Law** - relates latency, throughput, and concurrency
4. **Avoid high utilization** - latency spikes above 80%
5. **Tail latency matters** - one slow component slows everything
6. **Trade-offs exist** - batching increases throughput but adds latency

## Quick Reference

| Scenario | Optimize For | Strategy |
|----------|--------------|----------|
| User-facing API | Latency (p99) | Cache, CDN, async |
| Batch processing | Throughput | Batch, parallel |
| Real-time systems | Latency (p99.9) | Dedicated resources |
| Analytics | Throughput | Columnar storage, parallel |

## Practice Questions

1. Your API has p50=50ms but p99=2s. What's causing this? How would you fix it?
2. Calculate the servers needed for 100K RPS if each server handles 2K RPS.
3. Why does latency spike at high utilization?
4. How does adding a cache affect both latency and throughput?

## Further Reading

- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)
- [Thinking Clearly About Performance](https://queue.acm.org/detail.cfm?id=1854041)
- [The Tail at Scale](https://research.google/pubs/pub40801/)

---

Next: [Data Partitioning](../06-partitioning/README.md)
