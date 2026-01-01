# Numbers Cheat Sheet

Key numbers to memorize for system design interviews.

## Latency Numbers

```
L1 cache reference:                    0.5 ns
Branch mispredict:                     5 ns
L2 cache reference:                    7 ns
Mutex lock/unlock:                     25 ns
Main memory reference:                 100 ns
Compress 1KB with Zippy:               3,000 ns    (3 μs)
Send 1KB over 1 Gbps network:          10,000 ns   (10 μs)
Read 4KB randomly from SSD:            150,000 ns  (150 μs)
Read 1MB sequentially from memory:     250,000 ns  (250 μs)
Round trip within same datacenter:     500,000 ns  (500 μs)
Read 1MB sequentially from SSD:        1,000,000 ns (1 ms)
Disk seek:                             10,000,000 ns (10 ms)
Read 1MB sequentially from disk:       20,000,000 ns (20 ms)
Send packet CA → Netherlands → CA:     150,000,000 ns (150 ms)
```

## Time Conversions

```
1 second    = 1,000 milliseconds (ms)
1 second    = 1,000,000 microseconds (μs)
1 second    = 1,000,000,000 nanoseconds (ns)

1 minute    = 60 seconds
1 hour      = 3,600 seconds
1 day       = 86,400 seconds ≈ 100,000 seconds
1 month     = 2,592,000 seconds ≈ 2.5 million seconds
1 year      = 31,536,000 seconds ≈ 30 million seconds
```

## Data Size Units

```
1 Byte (B)        = 8 bits
1 Kilobyte (KB)   = 1,000 bytes
1 Megabyte (MB)   = 1,000 KB = 1,000,000 bytes
1 Gigabyte (GB)   = 1,000 MB = 1,000,000,000 bytes
1 Terabyte (TB)   = 1,000 GB
1 Petabyte (PB)   = 1,000 TB
```

## Common Data Sizes

```
UUID:                  16 bytes (128 bits)
Integer:               4-8 bytes
Timestamp:             8 bytes
Short string (50 chars): 50 bytes
Tweet/message:         100-500 bytes
Metadata record:       1 KB
Thumbnail image:       20-50 KB
Profile photo:         100-500 KB
Webpage:               2-5 MB
Photo (high res):      2-5 MB
Music file (MP3):      3-5 MB
Video (1 min, HD):     50-100 MB
Video (1 min, 4K):     300-500 MB
```

## Throughput Numbers

```
# Database operations (single instance)
PostgreSQL writes:      1,000-10,000/sec
PostgreSQL reads:       10,000-50,000/sec
MySQL writes:           1,000-10,000/sec
MySQL reads:            10,000-50,000/sec
Redis operations:       100,000+/sec
Cassandra writes:       10,000-50,000/sec per node

# Web servers
Single server requests: 10,000-100,000/sec (depends on operation)
WebSocket connections:  100,000-1,000,000 per server

# Network
1 Gbps:                 125 MB/sec
10 Gbps:                1.25 GB/sec
```

## Storage I/O

```
HDD:
  - Sequential read:    100-200 MB/sec
  - Random read:        100 IOPS
  - Latency:            10 ms

SSD:
  - Sequential read:    500-3000 MB/sec
  - Random read:        10,000-100,000 IOPS
  - Latency:            0.1 ms

NVMe SSD:
  - Sequential read:    3000-7000 MB/sec
  - Random read:        100,000-500,000 IOPS
  - Latency:            0.02 ms
```

## Availability Nines

```
Availability    Downtime/Year       Downtime/Month      Downtime/Week
99%             3.65 days           7.31 hours          1.68 hours
99.9%           8.77 hours          43.83 minutes       10.08 minutes
99.99%          52.60 minutes       4.38 minutes        1.01 minutes
99.999%         5.26 minutes        26.30 seconds       6.05 seconds
99.9999%        31.56 seconds       2.63 seconds        0.60 seconds
```

## Quick Estimation Rules

```
# Requests per second
1M requests/day ≈ 12 requests/sec
100M requests/day ≈ 1,200 requests/sec
1B requests/day ≈ 12,000 requests/sec

# Storage
1KB × 1M = 1 GB
1KB × 1B = 1 TB
1MB × 1M = 1 TB
1MB × 1B = 1 PB

# Users
1M DAU × 10 requests/user = 10M requests/day
100M DAU × 10 requests/user = 1B requests/day
```

## Typical Service Requirements

```
# Web application
Latency:        < 200 ms
Availability:   99.9% (8.77 hours downtime/year)

# Payment system
Latency:        < 1 second
Availability:   99.99% (52.60 minutes downtime/year)
Consistency:    Strong

# Real-time game
Latency:        < 50 ms
Availability:   99.9%

# Analytics dashboard
Latency:        < 5 seconds
Availability:   99%
```

## Memory/Caching Rules

```
80/20 Rule:     20% of data serves 80% of requests
Cache Size:     ~20% of total data for good hit rate
Cache Hit Rate: Target 90%+ for effective caching

# Redis capacity
1 node:         ~25 GB safe (with replication overhead)
Cluster:        Scale to terabytes
```

## Big Tech Scale Reference

```
Google Search:      ~100,000 queries/sec
Facebook:           ~600,000 queries/sec
Twitter:            ~6,000 tweets/sec
YouTube:            500 hours video uploaded/min
WhatsApp:           100 billion messages/day
Uber:               15 million trips/day
```

---

[Back to Resources](./README.md)
