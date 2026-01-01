# Back-of-Envelope Calculations

Quick estimations are essential in system design interviews. This chapter teaches you how to make reasonable approximations rapidly.

## Table of Contents
1. [Key Numbers to Know](#key-numbers-to-know)
2. [Traffic Estimation](#traffic-estimation)
3. [Storage Estimation](#storage-estimation)
4. [Bandwidth Estimation](#bandwidth-estimation)
5. [Example Calculations](#example-calculations)
6. [Interview Questions](#interview-questions)

---

## Key Numbers to Know

### Powers of 2

```
┌─────────────────────────────────────────────────────────────┐
│                    Powers of 2                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  2^10 = 1,024            ≈ 1 Thousand (KB)                  │
│  2^20 = 1,048,576        ≈ 1 Million (MB)                   │
│  2^30 = 1,073,741,824    ≈ 1 Billion (GB)                   │
│  2^40 = 1,099,511,627,776 ≈ 1 Trillion (TB)                 │
│                                                              │
│  Quick conversions:                                          │
│  1 KB = 2^10 bytes = 1,000 bytes                            │
│  1 MB = 2^20 bytes = 1,000,000 bytes                        │
│  1 GB = 2^30 bytes = 1,000,000,000 bytes                    │
│  1 TB = 2^40 bytes = 1,000,000,000,000 bytes                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Time Conversions

```
┌─────────────────────────────────────────────────────────────┐
│                  Time Conversions                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1 day    = 86,400 seconds    ≈ 100,000 seconds             │
│  1 month  = 2,500,000 seconds ≈ 2.5 million seconds         │
│  1 year   = 31,536,000 sec    ≈ 30 million seconds          │
│                                                              │
│  Quick trick for QPS:                                        │
│                                                              │
│  1 million requests/day                                     │
│  = 1,000,000 / 100,000                                      │
│  ≈ 10 requests/second                                       │
│                                                              │
│  Rule of thumb:                                              │
│  requests/day ÷ 100,000 = requests/second                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Common Data Sizes

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        Common Data Sizes                                    │
├──────────────────────────────────────┬─────────────────────────────────────┤
│ Item                                 │ Size                                │
├──────────────────────────────────────┼─────────────────────────────────────┤
│ Character (ASCII)                    │ 1 byte                              │
│ Character (UTF-8)                    │ 1-4 bytes                           │
│ Integer (32-bit)                     │ 4 bytes                             │
│ Long/Timestamp                       │ 8 bytes                             │
│ UUID                                 │ 16 bytes                            │
│ Tweet/Short message                  │ ~200 bytes                          │
│ Metadata record                      │ ~1 KB                               │
│ Thumbnail image                      │ ~20-50 KB                           │
│ Profile photo                        │ ~200-500 KB                         │
│ Web page                             │ ~2-5 MB                             │
│ High-res photo                       │ ~2-5 MB                             │
│ MP3 song                             │ ~3-5 MB                             │
│ 1 minute HD video                    │ ~50-100 MB                          │
│ 1 minute 4K video                    │ ~300-500 MB                         │
└──────────────────────────────────────┴─────────────────────────────────────┘
```

---

## Traffic Estimation

### Formula

```
┌─────────────────────────────────────────────────────────────┐
│                  Traffic Estimation                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: Identify users                                      │
│          Total users, DAU, MAU                              │
│                                                              │
│  Step 2: Estimate actions per user                          │
│          Reads/day, Writes/day                              │
│                                                              │
│  Step 3: Calculate total                                     │
│          Total = DAU × actions/user/day                     │
│                                                              │
│  Step 4: Convert to per second                              │
│          QPS = Total/day ÷ 86,400                           │
│                                                              │
│  Step 5: Consider peak                                       │
│          Peak = Average × 2-10x                             │
│                                                              │
│  Example:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ DAU: 10 million                                        │ │
│  │ Reads per user: 20/day                                 │ │
│  │ Writes per user: 2/day                                 │ │
│  │                                                        │ │
│  │ Reads/day  = 10M × 20 = 200M                          │ │
│  │ Writes/day = 10M × 2  = 20M                           │ │
│  │                                                        │ │
│  │ Read QPS  = 200M / 100K = 2,000/sec                   │ │
│  │ Write QPS = 20M / 100K  = 200/sec                     │ │
│  │                                                        │ │
│  │ Peak = 2-3x = 6,000 reads/sec, 600 writes/sec        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Storage Estimation

### Formula

```
┌─────────────────────────────────────────────────────────────┐
│                  Storage Estimation                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Step 1: Estimate object size                               │
│          Sum all fields in the record                       │
│                                                              │
│  Step 2: Estimate daily volume                              │
│          Objects/day × size                                 │
│                                                              │
│  Step 3: Calculate for retention period                     │
│          Daily × days retained                              │
│                                                              │
│  Step 4: Add overhead                                        │
│          Indexes, replication, overhead (~2-3x)             │
│                                                              │
│  Example (Twitter-like):                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Tweet size:                                            │ │
│  │   - tweet_id: 8 bytes                                  │ │
│  │   - user_id: 8 bytes                                   │ │
│  │   - content: 280 bytes                                 │ │
│  │   - timestamp: 8 bytes                                 │ │
│  │   - metadata: ~50 bytes                                │ │
│  │   Total: ~350 bytes ≈ 500 bytes (with padding)        │ │
│  │                                                        │ │
│  │ Tweets/day: 500 million                               │ │
│  │ Daily storage: 500M × 500B = 250 GB/day               │ │
│  │                                                        │ │
│  │ 5 years: 250 GB × 365 × 5 = 456 TB                   │ │
│  │ With 3x replication: ~1.4 PB                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Bandwidth Estimation

### Formula

```
┌─────────────────────────────────────────────────────────────┐
│                 Bandwidth Estimation                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Ingress (incoming) = Write QPS × Object size              │
│  Egress (outgoing)  = Read QPS × Object size               │
│                                                              │
│  Example (Image sharing):                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Upload: 10 images/sec × 2 MB = 20 MB/sec              │ │
│  │                              = 160 Mbps               │ │
│  │                                                        │ │
│  │ View: 1000 images/sec × 2 MB = 2 GB/sec               │ │
│  │                              = 16 Gbps                │ │
│  │                                                        │ │
│  │ With CDN: Most views from cache                       │ │
│  │ Origin bandwidth reduced by 90%+                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Network capacity:                                           │
│  1 Gbps = 125 MB/sec                                        │
│  10 Gbps = 1.25 GB/sec                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Example Calculations

### URL Shortener

```
┌─────────────────────────────────────────────────────────────┐
│              URL Shortener Estimation                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Assumptions:                                                │
│  - 100M new URLs/month                                      │
│  - Read:Write ratio = 100:1                                 │
│  - Keep for 5 years                                         │
│                                                              │
│  Traffic:                                                    │
│  - Writes: 100M/month = 100M/(30×24×3600) ≈ 40/sec         │
│  - Reads: 40 × 100 = 4,000/sec                             │
│                                                              │
│  Storage:                                                    │
│  - Short URL: 7 chars = 7 bytes                            │
│  - Long URL: average 100 bytes                             │
│  - Timestamp: 8 bytes                                       │
│  - Total: ~120 bytes per record                            │
│                                                              │
│  - URLs in 5 years: 100M × 12 × 5 = 6 billion             │
│  - Storage: 6B × 120B = 720 GB                             │
│  - With overhead (3x): ~2 TB                               │
│                                                              │
│  Bandwidth:                                                  │
│  - Write: 40/sec × 120B = 5 KB/sec                        │
│  - Read: 4000/sec × 120B = 480 KB/sec                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Video Streaming (YouTube-like)

```
┌─────────────────────────────────────────────────────────────┐
│             Video Streaming Estimation                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Assumptions:                                                │
│  - 1 billion DAU                                            │
│  - 5 videos watched/user/day                                │
│  - Average video: 5 minutes, 50 MB (compressed)            │
│  - 500K videos uploaded/day                                 │
│                                                              │
│  Traffic:                                                    │
│  - Views: 1B × 5 = 5B views/day                            │
│  - View QPS: 5B / 100K = 50,000/sec                        │
│  - Upload QPS: 500K / 100K = 5/sec                         │
│                                                              │
│  Storage (uploads):                                          │
│  - Raw upload: 5 min × 500 MB (original) = 2.5 GB/video   │
│  - Multiple resolutions: ~5 GB/video total                 │
│  - Daily: 500K × 5 GB = 2.5 PB/day                        │
│  - Yearly: 2.5 PB × 365 ≈ 900 PB = 0.9 EB                 │
│                                                              │
│  Bandwidth:                                                  │
│  - Viewing: 50K/sec × 50 MB / 300 sec = 8.3 TB/sec        │
│  - With CDN: 90% from edge = 830 GB/sec from origin        │
│                                                              │
│  Note: This is why YouTube uses massive CDN infrastructure │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│                Quick Estimation Cheat Sheet                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1M requests/day ≈ 10 QPS                                   │
│  100M requests/day ≈ 1,000 QPS                              │
│  1B requests/day ≈ 10,000 QPS                               │
│                                                              │
│  1 KB × 1M = 1 GB                                           │
│  1 KB × 1B = 1 TB                                           │
│  1 MB × 1M = 1 TB                                           │
│  1 MB × 1B = 1 PB                                           │
│                                                              │
│  Single server: ~10K-100K QPS (simple operations)           │
│  Single DB: ~10K QPS (depends on query complexity)          │
│  Redis: ~100K QPS                                           │
│                                                              │
│  Always mention:                                             │
│  - Peak vs average (2-10x)                                  │
│  - Growth over time                                         │
│  - Replication factor (usually 3x)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. How many seconds are in a day?
2. Estimate storage for 100 million user profiles.
3. Convert 10 million requests/day to QPS.

### Intermediate
4. Estimate storage needs for a photo sharing app.
5. Calculate bandwidth for a video streaming service.
6. How would you estimate Twitter's storage needs?

### Advanced
7. Design capacity for a global messaging app.
8. Estimate infrastructure for YouTube-scale video platform.
9. How would you size a distributed cache for 1B users?

---

[← Previous: Consensus](../08-consensus/README.md) | [Back to Module](../README.md) | [Next Module: Building Blocks →](../../03-building-blocks/README.md)
