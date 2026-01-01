# Estimation Techniques

Back-of-envelope calculations to justify your design decisions.

---

## Why Estimates Matter

```
┌─────────────────────────────────────────────────────────────┐
│              The Power of Estimation                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Estimates help you:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Validate design decisions                          │ │
│  │     "Can one server handle this load?"                 │ │
│  │     → Calculate QPS and compare to server capacity     │ │
│  │                                                         │ │
│  │  2. Identify bottlenecks                               │ │
│  │     "Where will we need to scale first?"               │ │
│  │     → Find components near capacity limits             │ │
│  │                                                         │ │
│  │  3. Choose appropriate technologies                    │ │
│  │     "Do we need a distributed database?"               │ │
│  │     → Calculate if data fits on one machine            │ │
│  │                                                         │ │
│  │  4. Show technical maturity                            │ │
│  │     Demonstrates real-world experience                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Rule: Estimates don't need to be exact!                   │
│  Being within 10x is usually good enough.                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Essential Numbers to Memorize

```
┌─────────────────────────────────────────────────────────────┐
│                Numbers You Should Know                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Time:                                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1 day    = 86,400 seconds ≈ 100,000 seconds          │ │
│  │  1 week   = 604,800 seconds ≈ 600,000 seconds         │ │
│  │  1 month  = 2.6M seconds ≈ 2.5M seconds               │ │
│  │  1 year   = 31.5M seconds ≈ 30M seconds               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Data Size:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1 Byte = 8 bits                                       │ │
│  │  1 KB = 1,000 bytes ≈ 1,000 chars (ASCII)             │ │
│  │  1 MB = 1,000 KB ≈ 1 minute of MP3                    │ │
│  │  1 GB = 1,000 MB ≈ 1 hour of HD video                 │ │
│  │  1 TB = 1,000 GB ≈ 1,000 hours of video               │ │
│  │  1 PB = 1,000 TB                                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Common Sizes:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Tweet (280 chars)          ≈ 280 bytes               │ │
│  │  Tweet with metadata        ≈ 1 KB                    │ │
│  │  Image (compressed)         ≈ 200 KB - 1 MB           │ │
│  │  Profile photo              ≈ 50-100 KB               │ │
│  │  1 minute video (720p)      ≈ 50 MB                   │ │
│  │  1 minute video (1080p)     ≈ 150 MB                  │ │
│  │  Database row (typical)     ≈ 100 bytes - 1 KB        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Server Capacity (rough):                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Web server (simple API)    ≈ 1,000 QPS               │ │
│  │  Web server (complex logic) ≈ 100-500 QPS             │ │
│  │  Database (MySQL/Postgres)  ≈ 10,000 QPS (reads)      │ │
│  │  Cache (Redis)              ≈ 100,000 QPS             │ │
│  │  SSD read                   ≈ 100,000 IOPS            │ │
│  │  HDD read                   ≈ 100-200 IOPS            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Latency:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  L1 cache reference         ≈ 1 ns                    │ │
│  │  L2 cache reference         ≈ 4 ns                    │ │
│  │  RAM reference              ≈ 100 ns                  │ │
│  │  SSD random read            ≈ 100 μs                  │ │
│  │  HDD seek                   ≈ 10 ms                   │ │
│  │  Same datacenter RTT        ≈ 0.5 ms                  │ │
│  │  Cross-country RTT          ≈ 50-100 ms               │ │
│  │  Cross-Atlantic RTT         ≈ 100-150 ms              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## QPS (Queries Per Second) Calculations

```
┌─────────────────────────────────────────────────────────────┐
│                  QPS Calculations                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Formula:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  QPS = (Daily Active Users × Actions per User)         │ │
│  │        ─────────────────────────────────────────        │ │
│  │                  Seconds per Day                        │ │
│  │                                                         │ │
│  │  Seconds per day = 86,400 ≈ 100,000 (for easy math)   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - Twitter:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 200M DAU                                            │ │
│  │  • Average user views 50 tweets/day                    │ │
│  │  • Average user posts 0.5 tweets/day                   │ │
│  │                                                         │ │
│  │  Read QPS:                                              │ │
│  │  = (200M × 50) / 100,000                               │ │
│  │  = 10B / 100,000                                        │ │
│  │  = 100,000 QPS                                          │ │
│  │                                                         │ │
│  │  Write QPS:                                             │ │
│  │  = (200M × 0.5) / 100,000                              │ │
│  │  = 100M / 100,000                                       │ │
│  │  = 1,000 QPS                                            │ │
│  │                                                         │ │
│  │  Peak QPS = Average × 3 (typical peak factor)          │ │
│  │  Peak Read QPS = 300,000 QPS                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Server Sizing:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  If each server handles 1,000 QPS:                     │ │
│  │                                                         │ │
│  │  Servers needed = Peak QPS / Capacity per server       │ │
│  │                 = 300,000 / 1,000                       │ │
│  │                 = 300 servers                           │ │
│  │                                                         │ │
│  │  Add buffer (usually 2-3x for redundancy):             │ │
│  │  = 300 × 2 = 600 servers                               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Storage Calculations

```
┌─────────────────────────────────────────────────────────────┐
│                 Storage Calculations                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Formula:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Daily Storage = New Items × Size per Item             │ │
│  │  Yearly Storage = Daily Storage × 365                  │ │
│  │  Total Storage = Yearly Storage × Years × Replication  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - Twitter (Text Only):                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 500M tweets/day                                     │ │
│  │  • Tweet size = 280 bytes text + 500 bytes metadata    │ │
│  │  • Total = ~1 KB per tweet                             │ │
│  │                                                         │ │
│  │  Daily storage:                                         │ │
│  │  = 500M × 1 KB                                          │ │
│  │  = 500 GB/day                                           │ │
│  │                                                         │ │
│  │  Yearly storage:                                        │ │
│  │  = 500 GB × 365                                         │ │
│  │  = 180 TB/year                                          │ │
│  │                                                         │ │
│  │  5 years with 3x replication:                          │ │
│  │  = 180 TB × 5 × 3                                       │ │
│  │  = 2.7 PB                                               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - Instagram (Images):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 100M photos/day                                     │ │
│  │  • Average photo = 500 KB                              │ │
│  │  • Store 3 sizes (thumbnail, medium, original)         │ │
│  │  • Total = 500 KB × 3 = 1.5 MB per photo               │ │
│  │                                                         │ │
│  │  Daily storage:                                         │ │
│  │  = 100M × 1.5 MB                                        │ │
│  │  = 150 TB/day                                           │ │
│  │                                                         │ │
│  │  Yearly storage:                                        │ │
│  │  = 150 TB × 365                                         │ │
│  │  = 55 PB/year                                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Bandwidth Calculations

```
┌─────────────────────────────────────────────────────────────┐
│                 Bandwidth Calculations                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Formula:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Bandwidth = QPS × Average Response Size               │ │
│  │                                                         │ │
│  │  Note: Calculate separately for:                       │ │
│  │  • Ingress (data coming in)                            │ │
│  │  • Egress (data going out)                             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - YouTube:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 1B daily video views                                │ │
│  │  • Average video: 5 minutes at 720p                    │ │
│  │  • 720p bitrate: ~3 Mbps = 375 KB/s                    │ │
│  │  • 5 min = 300 seconds × 375 KB = 112 MB per view      │ │
│  │                                                         │ │
│  │  Daily egress:                                          │ │
│  │  = 1B × 112 MB                                          │ │
│  │  = 112 PB/day                                           │ │
│  │                                                         │ │
│  │  Peak bandwidth (assume 3x average):                   │ │
│  │  = (112 PB / 86,400 seconds) × 3                       │ │
│  │  = 1.3 PB/s × 3                                         │ │
│  │  = 3.9 PB/s                                             │ │
│  │                                                         │ │
│  │  This is why YouTube needs massive CDN infrastructure! │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - API Service:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 50,000 QPS peak                                     │ │
│  │  • Average request: 1 KB                               │ │
│  │  • Average response: 10 KB                             │ │
│  │                                                         │ │
│  │  Ingress: 50,000 × 1 KB = 50 MB/s                      │ │
│  │  Egress: 50,000 × 10 KB = 500 MB/s = 4 Gbps            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Memory Calculations

```
┌─────────────────────────────────────────────────────────────┐
│                  Memory Calculations                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Cache Sizing:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Rule of thumb: Cache 20% of most accessed data        │ │
│  │  (Pareto principle: 20% of data serves 80% of requests)│ │
│  │                                                         │ │
│  │  Example - Twitter Timeline Cache:                     │ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 200M DAU                                            │ │
│  │  • Cache last 800 tweets per user                      │ │
│  │  • Each tweet ID = 8 bytes                             │ │
│  │                                                         │ │
│  │  Cache size:                                            │ │
│  │  = 200M users × 800 tweets × 8 bytes                   │ │
│  │  = 1.28 TB                                              │ │
│  │                                                         │ │
│  │  Redis servers (64GB each):                            │ │
│  │  = 1.28 TB / 64 GB                                      │ │
│  │  = 20 servers (plus replication)                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Session Storage:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given:                                                 │ │
│  │  • 10M concurrent users                                │ │
│  │  • Session size = 2 KB                                 │ │
│  │                                                         │ │
│  │  Memory needed:                                         │ │
│  │  = 10M × 2 KB                                           │ │
│  │  = 20 GB                                                │ │
│  │                                                         │ │
│  │  Fits on a single large Redis instance!                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Practice Problems

```
┌─────────────────────────────────────────────────────────────┐
│                   Practice Problems                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem 1: URL Shortener                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Given:                                                 │ │
│  │  • 100M new URLs/month                                 │ │
│  │  • 100:1 read/write ratio                              │ │
│  │  • Store URLs for 5 years                              │ │
│  │  • Average URL = 100 bytes                             │ │
│  │                                                         │ │
│  │  Calculate:                                             │ │
│  │  1. Write QPS                                          │ │
│  │  2. Read QPS                                           │ │
│  │  3. Total storage                                      │ │
│  │                                                         │ │
│  │  Solution:                                              │ │
│  │  1. Write QPS = 100M / (30 × 86,400) ≈ 40 QPS         │ │
│  │  2. Read QPS = 40 × 100 = 4,000 QPS                   │ │
│  │  3. Storage = 100M × 12 months × 5 years × 100 bytes  │ │
│  │             = 6B × 100 bytes = 600 GB                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Problem 2: Chat Application                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Given:                                                 │ │
│  │  • 50M DAU                                             │ │
│  │  • Each user sends 20 messages/day                     │ │
│  │  • Each user receives 40 messages/day                  │ │
│  │  • Message size = 200 bytes                            │ │
│  │                                                         │ │
│  │  Calculate:                                             │ │
│  │  1. Messages sent per day                              │ │
│  │  2. Write QPS                                          │ │
│  │  3. Daily storage                                      │ │
│  │                                                         │ │
│  │  Solution:                                              │ │
│  │  1. Messages = 50M × 20 = 1B messages/day             │ │
│  │  2. Write QPS = 1B / 86,400 ≈ 12,000 QPS              │ │
│  │  3. Storage = 1B × 200 bytes = 200 GB/day             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Problem 3: Video Streaming                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Given:                                                 │ │
│  │  • 10M concurrent viewers                              │ │
│  │  • 1080p stream = 5 Mbps                               │ │
│  │  • 50% watch 1080p, 50% watch 720p (2.5 Mbps)          │ │
│  │                                                         │ │
│  │  Calculate:                                             │ │
│  │  1. Total bandwidth needed                             │ │
│  │                                                         │ │
│  │  Solution:                                              │ │
│  │  1. 1080p: 5M × 5 Mbps = 25 Tbps                       │ │
│  │     720p: 5M × 2.5 Mbps = 12.5 Tbps                    │ │
│  │     Total = 37.5 Tbps                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│              Estimation Cheat Sheet                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Conversions:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1 day ≈ 100,000 seconds                               │ │
│  │  1 year ≈ 30M seconds                                  │ │
│  │  1 million ≈ 10^6                                      │ │
│  │  1 billion ≈ 10^9                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  QPS:                                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  QPS = Daily Requests / 100,000                        │ │
│  │  Peak QPS = Average QPS × 3                            │ │
│  │  Servers = Peak QPS / Server Capacity                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Storage:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Daily = Items × Size                                  │ │
│  │  Total = Daily × 365 × Years × Replication             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Bandwidth:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bandwidth = QPS × Response Size                       │ │
│  │  1 Gbps ≈ 125 MB/s                                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Cache:                                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Cache ~20% of data for 80% hit rate                   │ │
│  │  Redis: ~100K QPS per instance                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Back to Module](./README.md) | [Next: Communication Tips →](./04-communication-tips.md)
