# Back-of-envelope Calculations

The ability to quickly estimate system capacity is a crucial skill for system design interviews. This chapter provides the tools and techniques for rapid estimation.

## Table of Contents
- [Why Estimate?](#why-estimate)
- [Essential Numbers](#essential-numbers)
- [Common Calculations](#common-calculations)
- [Estimation Frameworks](#estimation-frameworks)
- [Practice Problems](#practice-problems)
- [Key Takeaways](#key-takeaways)

## Why Estimate?

In system design interviews, you need to:
- Justify design decisions with data
- Identify bottlenecks before they appear
- Choose appropriate technologies
- Plan capacity for growth

**The goal isn't precision** - it's getting within an order of magnitude.

## Essential Numbers

### Powers of 2

```
┌─────────────────────────────────────────────────────────┐
│                   Powers of 2                           │
├─────────────────────────────────────────────────────────┤
│  2^10 = 1,024           ≈ 1 Thousand (1 KB)            │
│  2^20 = 1,048,576       ≈ 1 Million (1 MB)             │
│  2^30 = 1,073,741,824   ≈ 1 Billion (1 GB)             │
│  2^40 = 1,099,511,627,776 ≈ 1 Trillion (1 TB)          │
└─────────────────────────────────────────────────────────┘
```

### Time Conversions

```
┌─────────────────────────────────────────────────────────┐
│                 Time Conversions                        │
├─────────────────────────────────────────────────────────┤
│  1 day    = 86,400 seconds     ≈ 100,000 = 10^5        │
│  1 week   = 604,800 seconds    ≈ 600,000 = 6 × 10^5    │
│  1 month  = 2,592,000 seconds  ≈ 2.5 × 10^6            │
│  1 year   = 31,536,000 seconds ≈ 3 × 10^7              │
└─────────────────────────────────────────────────────────┘
```

### Latency Numbers

```
┌─────────────────────────────────────────────────────────┐
│         Latency Numbers Every Programmer Should Know    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  L1 cache reference                    0.5 ns           │
│  Branch mispredict                     5   ns           │
│  L2 cache reference                    7   ns           │
│  Mutex lock/unlock                     25  ns           │
│  Main memory reference                 100 ns           │
│  Compress 1K bytes with Zippy          3,000 ns = 3 μs  │
│  Send 1K bytes over 1 Gbps network     10,000 ns = 10 μs│
│  Read 4K randomly from SSD             150,000 ns = 150 μs
│  Read 1 MB sequentially from memory    250,000 ns = 250 μs
│  Round trip within same datacenter     500,000 ns = 0.5 ms
│  Read 1 MB sequentially from SSD       1,000,000 ns = 1 ms
│  Disk seek                             10,000,000 ns = 10 ms
│  Read 1 MB sequentially from disk      20,000,000 ns = 20 ms
│  Send packet CA→Netherlands→CA         150,000,000 ns = 150 ms
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Throughput Numbers

```
┌─────────────────────────────────────────────────────────┐
│                 Throughput Numbers                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  HDD:                                                   │
│    Sequential read/write: 100-200 MB/s                  │
│    Random IOPS: 100-200                                │
│                                                         │
│  SSD:                                                   │
│    Sequential read: 500-3000 MB/s                       │
│    Sequential write: 400-2000 MB/s                      │
│    Random IOPS: 50,000-500,000                         │
│                                                         │
│  Network:                                               │
│    1 Gbps = 125 MB/s                                   │
│    10 Gbps = 1.25 GB/s                                 │
│    100 Gbps = 12.5 GB/s                                │
│                                                         │
│  Database (single node):                                │
│    PostgreSQL: 10,000-50,000 QPS                       │
│    MySQL: 10,000-50,000 QPS                            │
│    Redis: 100,000-500,000 QPS                          │
│                                                         │
│  Web server:                                            │
│    Simple API: 1,000-10,000 RPS per server             │
│    Static content: 10,000-100,000 RPS per server       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Common Data Sizes

```
┌─────────────────────────────────────────────────────────┐
│                   Data Sizes                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Text:                                                  │
│    Tweet (280 chars): ~0.5 KB                          │
│    Email: ~50 KB average                               │
│    Web page HTML: ~100 KB                              │
│                                                         │
│  Media:                                                 │
│    Thumbnail: 10-50 KB                                 │
│    Standard image: 200-500 KB                          │
│    High-res image: 2-5 MB                              │
│    1 min video (compressed): 5-10 MB                   │
│    1 min video (HD): 50-100 MB                         │
│                                                         │
│  User data:                                             │
│    User profile: 1-10 KB                               │
│    Session: 1-5 KB                                     │
│    Shopping cart: 5-20 KB                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Common Calculations

### QPS (Queries Per Second)

```
Formula: QPS = Daily Active Users × Actions per User / Seconds per Day

Example: Twitter-like service
- 300 million DAU
- Average user views 100 tweets/day
- Seconds per day: 86,400 ≈ 100,000

Read QPS = 300M × 100 / 100,000 = 300,000 QPS

Peak QPS = 2-3× average = 600,000 - 900,000 QPS
```

### Storage

```
Formula: Storage = Number of Objects × Size per Object × Retention Period

Example: Photo sharing service
- 500 million users
- 1 photo/user/day average
- Photo size: 500 KB
- Retention: Forever (5 years for calculation)

Daily new storage = 500M × 500 KB = 250 TB/day
5-year storage = 250 TB × 365 × 5 = 456 PB
```

### Bandwidth

```
Formula: Bandwidth = QPS × Data Size per Request

Example: Video streaming
- 1 million concurrent viewers
- Bitrate: 5 Mbps per stream

Bandwidth = 1M × 5 Mbps = 5 Tbps

Egress cost at $0.05/GB:
Per hour = 5 Tbps × 3600s / 8 = 2.25 PB/hour
Cost = 2.25 PB × 1000 × $0.05 = $112,500/hour
```

### Servers Needed

```
Formula: Servers = Peak QPS / QPS per Server

Example: API service
- Peak QPS: 100,000
- Single server handles: 5,000 QPS

Servers = 100,000 / 5,000 = 20 servers

With redundancy (N+2): 22 servers
Across 3 availability zones: 24 servers (8 per zone)
```

## Estimation Frameworks

### Step-by-Step Approach

```
1. Clarify Requirements
   - Daily/Monthly active users
   - Read/Write ratio
   - Data types and sizes

2. Traffic Estimation
   - QPS (average and peak)
   - Read vs Write QPS

3. Storage Estimation
   - Per-object size
   - Retention period
   - Growth rate

4. Bandwidth Estimation
   - Ingress (uploads)
   - Egress (downloads)

5. Infrastructure Estimation
   - Servers needed
   - Database capacity
   - Cache size
```

### Example: URL Shortener

```
Requirements:
- 500 million new URLs per month
- 100:1 read:write ratio
- Store for 5 years
- URL data: 500 bytes average

Traffic:
- Write: 500M / (30 × 24 × 3600) = 200 URLs/sec
- Read: 200 × 100 = 20,000 URLs/sec
- Peak: 40,000 reads/sec

Storage:
- Per month: 500M × 500 bytes = 250 GB
- 5 years: 250 GB × 12 × 5 = 15 TB

Bandwidth:
- Ingress: 200 × 500 bytes = 100 KB/s (negligible)
- Egress: 20,000 × 500 bytes = 10 MB/s

Cache:
- 20% of URLs get 80% of traffic
- Cache 20% of 5-year data: 15 TB × 0.2 = 3 TB
- With 80% hit rate: Memory for 3 TB

Servers:
- 40K reads/sec ÷ 10K/server = 4 read servers
- Add redundancy: 6-8 servers minimum
```

### Example: Instagram-like Service

```
Requirements:
- 500 million DAU
- 2 photos uploaded per user per day (active users: 10%)
- 200 photos viewed per user per day
- Photo size: 500 KB average
- Store forever

Traffic:
- Photo uploads: 500M × 0.1 × 2 / 86400 = 1,200/sec
- Photo views: 500M × 200 / 86400 = 1.2M/sec

Storage:
- Daily: 500M × 0.1 × 2 × 500 KB = 50 TB/day
- Yearly: 50 TB × 365 = 18 PB/year
- Multiple sizes (thumbnail, medium, full): 18 PB × 3 = 54 PB/year

Bandwidth:
- Upload: 1,200 × 500 KB = 600 MB/s
- Download: 1.2M × 100 KB (served size) = 120 GB/s = 960 Gbps

CDN:
- Offload 90% of reads to CDN
- Origin serves 10%: 96 Gbps

Database:
- 100M new photos/day metadata
- Photo record: 200 bytes
- Daily metadata: 20 GB
- Yearly: 7 TB
```

## Rounding Rules

### Keep It Simple

```
Use powers of 10:
- 86,400 seconds/day → 100,000 (10^5)
- 2.6 million seconds/month → 3 million (3 × 10^6)

Round to convenient numbers:
- 347 million users → 300 million or 400 million
- 8,756 → 10,000
- 0.23 → 0.25 (or 25%)
```

### Error Tolerance

```
Goal: Within 10× of actual (one order of magnitude)

Example:
Actual: 127,453 QPS
Your estimate: 100,000 QPS
Error: ~20% - Acceptable!

Actual: 127,453 QPS
Your estimate: 10,000 QPS
Error: 10× - Too far off!
```

## Practice Problems

### Problem 1: Twitter Feed

```
Given:
- 400 million MAU, 200 million DAU
- 50 million tweets per day
- Average user follows 200 accounts
- Average user reads 100 tweets per session
- 2 sessions per day

Calculate:
1. Write QPS (tweets)
2. Read QPS (feed loads)
3. Storage per day
4. Fan-out on write (notifications)

Answers:
1. 50M / 86400 ≈ 600 tweets/sec
2. 200M × 2 × 100 / 86400 ≈ 400K reads/sec
3. 50M × 1KB = 50 GB/day (text + metadata)
4. 50M × 200 (avg followers) = 10B fan-out operations/day
```

### Problem 2: YouTube

```
Given:
- 2 billion MAU
- 1 billion hours watched per day
- 500 hours of video uploaded per minute
- Average video: 10 minutes, 50 MB

Calculate:
1. Upload bandwidth
2. Download bandwidth
3. Storage per day
4. CDN requirements

Answers:
1. 500 × 60 × 50 MB = 1.5 TB/hour = 400 MB/s upload
2. 1B hours × 60 min × 5 MB/min / 86400 = 3.5 TB/s
3. 500 × 60 × 24 × 50 MB = 36 TB/day (raw)
   With transcoding (5 qualities): 180 TB/day
4. 3.5 TB/s = 28 Tbps (need global CDN)
```

### Problem 3: Ride-Sharing (Uber)

```
Given:
- 100 million riders
- 5 million drivers
- 20 million rides per day
- Location update every 4 seconds (drivers)

Calculate:
1. Ride requests QPS
2. Location updates QPS
3. Storage for location data (1 month)
4. Matching computation load

Answers:
1. 20M / 86400 ≈ 230 requests/sec
2. 5M active drivers × (1/4) = 1.25M updates/sec
3. 1.25M × 100 bytes × 86400 × 30 = 324 TB/month
4. Each request checks ~100 nearby drivers
   230 × 100 = 23,000 comparisons/sec
```

## Quick Reference Formulas

```
┌─────────────────────────────────────────────────────────┐
│                Quick Reference                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  QPS = DAU × requests_per_user / 86,400                │
│                                                         │
│  Storage = objects × size × retention                   │
│                                                         │
│  Bandwidth = QPS × data_per_request                     │
│                                                         │
│  Servers = peak_QPS / QPS_per_server                   │
│                                                         │
│  Cache size = working_set × (1 / hit_rate_needed)      │
│                                                         │
│  Replication factor = 3 (typical)                      │
│  Storage with replication = raw_storage × RF           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Memorize key numbers** - powers of 2, latencies, throughput
2. **Round aggressively** - precision isn't the goal
3. **Show your work** - interviewers want to see reasoning
4. **Start with traffic** - then storage, then bandwidth
5. **Account for growth** - design for 10× current load
6. **Consider peaks** - usually 2-3× average

## Further Reading

- [Napkin Math](https://github.com/sirupsen/napern-math)
- [System Design Primer - Back of Envelope](https://github.com/donnemartin/system-design-primer#back-of-the-envelope-calculations)
- [Numbers Everyone Should Know](http://highscalability.com/blog/2011/1/26/google-pro-tip-use-back-of-the-envelope-calculations-to-choo.html)

---

[Back to Module 2](../README.md) | Next: [Module 3: Building Blocks](../../03-building-blocks/README.md)
