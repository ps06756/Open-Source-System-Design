# Capacity Estimation

Back-of-envelope calculations to understand the scale of your system.

## Why Estimate?

- Guides architectural decisions
- Identifies bottlenecks early
- Shows you can think quantitatively
- Helps justify technology choices

## Key Numbers to Know

### Time Units
```
1 day = 86,400 seconds ≈ 100,000 (10^5) seconds
1 month ≈ 2.5 million seconds
1 year ≈ 30 million seconds
```

### Data Sizes
```
1 KB = 1,000 bytes (text, small JSON)
1 MB = 1,000 KB (image, audio clip)
1 GB = 1,000 MB (video, large dataset)
1 TB = 1,000 GB (database)
1 PB = 1,000 TB (data warehouse)
```

### Common Item Sizes
```
Tweet/message: 100-500 bytes
Metadata record: 1 KB
Thumbnail image: 20 KB
Profile photo: 200 KB
Regular image: 2 MB
Video (1 min): 100 MB
```

### Throughput
```
Single server: 10K-100K requests/sec (depends on operation)
Database write: 1K-10K writes/sec
Database read: 10K-100K reads/sec
SSD random read: 100K IOPS
HDD random read: 100 IOPS
```

## Estimation Framework

### Step 1: Traffic Estimation

```
Given:
- Daily Active Users (DAU)
- Actions per user per day

Calculate:
- Writes/day = DAU × writes/user
- Reads/day = DAU × reads/user
- Writes/sec = Writes/day ÷ 86,400
- Reads/sec = Reads/day ÷ 86,400
- Peak (2-5× average)
```

### Step 2: Storage Estimation

```
Calculate:
- Data per item × items/day × retention days
- Include all data types (main data, metadata, logs)
- Account for replication (2-3×)
```

### Step 3: Bandwidth Estimation

```
Calculate:
- Ingress: Writes/sec × size per write
- Egress: Reads/sec × size per read
- Consider: Do reads serve cached content?
```

## Example: Design Instagram

### Given
- 500M DAU
- 100M photos uploaded/day
- 10 billion photo views/day

### Traffic Estimation

```
Uploads:
- 100M photos/day
- 100M ÷ 86,400 ≈ 1,200 uploads/sec
- Peak: 3,000 uploads/sec

Views:
- 10B views/day
- 10B ÷ 86,400 ≈ 115,000 views/sec
- Peak: 300,000 views/sec

Read/Write ratio: 115,000 / 1,200 ≈ 100:1
→ Very read-heavy, caching will be important
```

### Storage Estimation

```
Photo storage:
- Average photo: 2 MB (after compression)
- 100M photos/day × 2 MB = 200 TB/day
- Per year: 200 TB × 365 = 73 PB/year
- With thumbnails (4 sizes): ~100 PB/year

Metadata storage:
- 1 KB per photo × 100M/day = 100 GB/day
- Per year: 36 TB (much smaller)

Total: ~100 PB/year for images
→ Need object storage (S3), not traditional DB
```

### Bandwidth Estimation

```
Ingress (uploads):
- 1,200/sec × 2 MB = 2.4 GB/sec

Egress (views):
- Assume thumbnail (100 KB) 80% of time
- Assume full (500 KB) 20% of time
- Average: 100 KB × 0.8 + 500 KB × 0.2 = 180 KB
- 115,000/sec × 180 KB = 20 GB/sec
- With CDN cache hits (90%): 2 GB/sec from origin

→ CDN essential, 90% cache hit saves 18 GB/sec
```

## Quick Estimation Shortcuts

### Users to Requests
```
Light usage: 1-5 requests/user/day
Medium: 10-50 requests/user/day
Heavy: 100+ requests/user/day
```

### Servers Needed
```
Requests/sec ÷ 10,000 = rough server count
(Very rough - depends on operation complexity)
```

### Cache Size
```
80/20 rule: 20% of data serves 80% of requests
Cache size = Total data × 0.2
```

## Example: URL Shortener

### Given
- 100M new URLs/month
- 10B redirects/month

### Calculations

```
Traffic:
- Writes: 100M/month = 100M ÷ (30 × 86,400) ≈ 40/sec
- Reads: 10B/month = 10B ÷ (30 × 86,400) ≈ 4,000/sec
- Read/Write: 100:1

Storage (5 years):
- URLs: 100M × 12 × 5 = 6B URLs
- Per URL: ~500 bytes (short code + long URL + metadata)
- Total: 6B × 500 bytes = 3 TB
→ Fits on a single machine, but shard for availability

Cache:
- 20% hot URLs: 6B × 0.2 = 1.2B URLs
- Size: 1.2B × 500 bytes = 600 GB
→ Need distributed cache (Redis cluster)
```

## Common Mistakes

### 1. Over-Precision
Don't calculate to exact numbers. "About 100,000 requests/sec" is fine.

### 2. Forgetting Peak Traffic
Average is not enough. Systems must handle 2-5× average for peaks.

### 3. Ignoring Replication
Data is usually replicated 2-3×. Storage estimates should account for this.

### 4. Not Sanity Checking
Does your answer make sense? 1 PB/day for a simple app is probably wrong.

## Practice Problems

1. **YouTube**: 2B users, 1B videos, 500 hours uploaded/minute
2. **WhatsApp**: 2B users, 100B messages/day
3. **Uber**: 100M users, 15M rides/day, location updates every 4 seconds

---

Next: [High-Level Design](../04-high-level-design/README.md)
