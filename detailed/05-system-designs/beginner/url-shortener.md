# Design URL Shortener (like bit.ly)

## 1. Requirements

### Functional Requirements
- Given a long URL, generate a short URL
- When user visits short URL, redirect to original URL
- Optional: Custom short URLs
- Optional: Analytics (click count)
- Optional: URL expiration

### Non-Functional Requirements
- High availability (redirects should always work)
- Low latency for redirects (< 100ms)
- Short URLs should be as short as possible
- Non-predictable URLs (security)

### Scale Estimates
- 100M new URLs per month
- 10:1 read to write ratio = 1B redirects per month
- URLs stored for 5 years = 6B total URLs
- Average URL length: 100 bytes
- Storage: 6B × 100 bytes = 600 GB

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Writes: 100M/month = ~40 URLs/second                      │
│  Reads: 1B/month = ~400 redirects/second                   │
│  Storage: ~600 GB over 5 years                             │
│  Read-heavy workload: 10:1 ratio                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  URL Shortening:                                            │
│  ┌────────┐    POST /shorten    ┌─────────────┐            │
│  │ Client │───────────────────▶│ API Server  │            │
│  └────────┘                     └──────┬──────┘            │
│                                        │                    │
│                    ┌───────────────────┼───────────────────┐│
│                    ▼                   ▼                   ││
│             ┌───────────┐       ┌───────────┐             ││
│             │  Key Gen  │       │ Database  │             ││
│             │  Service  │       │           │             ││
│             └───────────┘       └───────────┘             ││
│                                                              │
│  URL Redirect:                                               │
│  ┌────────┐   GET /abc123    ┌───────────┐    ┌─────────┐  │
│  │ Client │─────────────────▶│ API Server│───▶│  Cache  │  │
│  └────────┘                  └─────┬─────┘    └────┬────┘  │
│       ▲                           │               │         │
│       │                           │    Miss       ▼         │
│       │   HTTP 302               │         ┌─────────┐     │
│       └───────────────────────────┘         │ Database│     │
│                                             └─────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### API Design

```
POST /api/v1/shorten
Request:
{
    "long_url": "https://example.com/very/long/path",
    "custom_alias": "mylink",  // optional
    "expires_at": "2025-01-01"  // optional
}
Response:
{
    "short_url": "https://short.url/abc123",
    "expires_at": "2025-01-01"
}

GET /{short_code}
Response: HTTP 302 Redirect to long_url
Headers:
  Location: https://example.com/very/long/path
```

---

## 3. Key Generation Approaches

### Approach 1: Hash + Truncate

```
┌─────────────────────────────────────────────────────────────┐
│                  Hash + Truncate                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  long_url = "https://example.com/very/long/path"           │
│       │                                                      │
│       ▼                                                      │
│  MD5/SHA256 hash                                            │
│       │                                                      │
│       ▼                                                      │
│  "5eb63bbbe01eeed093cb22bb8f5acdc3"                        │
│       │                                                      │
│       ▼                                                      │
│  Take first 7 characters: "5eb63bb"                        │
│       │                                                      │
│       ▼                                                      │
│  Convert to Base62: "abc123X"                              │
│                                                              │
│  Problem: Collisions!                                       │
│  Same input → same hash → need collision handling          │
│  Solution: Append counter or user_id before hashing        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Approach 2: Base62 Counter (Recommended)

```
┌─────────────────────────────────────────────────────────────┐
│                 Base62 Counter                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Base62 = [0-9, a-z, A-Z] = 62 characters                  │
│                                                              │
│  7 character code = 62^7 = 3.5 trillion combinations!      │
│                                                              │
│  Counter to Base62:                                         │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Counter    Base62                                      ││
│  │  0          0000000                                     ││
│  │  1          0000001                                     ││
│  │  61         000000Z                                     ││
│  │  62         0000010                                     ││
│  │  1000000    00004c92                                    ││
│  └────────────────────────────────────────────────────────┘│
│                                                              │
│  How to generate unique counter?                           │
│                                                              │
│  Option A: Single counter (bottleneck)                     │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Counter Service (single)                               ││
│  │  counter++ → 1, 2, 3, 4...                             ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Option B: Range-based (recommended)                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Key Generation Service                                 ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐             ││
│  │  │ Server 1 │  │ Server 2 │  │ Server 3 │             ││
│  │  │ 1-1000   │  │1001-2000 │  │2001-3000 │             ││
│  │  └──────────┘  └──────────┘  └──────────┘             ││
│  │                                                         ││
│  │  When range exhausted, request new range from DB       ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Approach 3: Pre-generated Keys

```
┌─────────────────────────────────────────────────────────────┐
│                Pre-generated Keys                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Key Database                                           ││
│  │  ┌─────────────────┬────────────────────────┐          ││
│  │  │ Unused Keys     │ Used Keys               │          ││
│  │  ├─────────────────┼────────────────────────┤          ││
│  │  │ abc123          │ xyz789 → http://...    │          ││
│  │  │ def456          │ uvw456 → http://...    │          ││
│  │  │ ghi789          │                         │          ││
│  │  │ ...             │                         │          ││
│  │  └─────────────────┴────────────────────────┘          ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Background job generates keys in advance                   │
│                                                              │
│  On request:                                                │
│  1. Take key from unused pool (atomic operation)           │
│  2. Move to used pool with mapping                         │
│                                                              │
│  ✓ No collision risk                                       │
│  ✓ Fast key allocation                                     │
│  ✗ Requires pre-generation                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Table: urls                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ short_code (PK)  │ VARCHAR(7)                          │ │
│  │ long_url         │ VARCHAR(2048)                       │ │
│  │ user_id          │ BIGINT (nullable)                   │ │
│  │ created_at       │ TIMESTAMP                           │ │
│  │ expires_at       │ TIMESTAMP (nullable)                │ │
│  │ click_count      │ BIGINT DEFAULT 0                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Indexes:                                                    │
│  • PRIMARY KEY (short_code)                                │
│  • INDEX (user_id) - for user's URLs                       │
│  • INDEX (expires_at) - for cleanup job                    │
│                                                              │
│  Table: analytics (optional)                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ id              │ BIGINT AUTO_INCREMENT                │ │
│  │ short_code (FK) │ VARCHAR(7)                           │ │
│  │ timestamp       │ TIMESTAMP                            │ │
│  │ user_agent      │ VARCHAR(512)                         │ │
│  │ ip_address      │ VARCHAR(45)                          │ │
│  │ referrer        │ VARCHAR(2048)                        │ │
│  │ country         │ VARCHAR(2)                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Database choice:                                           │
│  • NoSQL (DynamoDB, Cassandra): Simple key-value, scales  │
│  • SQL (PostgreSQL, MySQL): If need complex queries       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Detailed Design

```
┌─────────────────────────────────────────────────────────────┐
│                   Detailed Architecture                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Load Balancer                        │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│           ┌───────────────┼───────────────┐                 │
│           ▼               ▼               ▼                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ API Server  │ │ API Server  │ │ API Server  │           │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘           │
│         │               │               │                   │
│         └───────────────┼───────────────┘                   │
│                         │                                    │
│         ┌───────────────┴───────────────┐                   │
│         ▼                               ▼                   │
│  ┌─────────────┐                 ┌─────────────┐           │
│  │   Redis     │                 │ Key Gen     │           │
│  │   Cache     │                 │ Service     │           │
│  └──────┬──────┘                 └──────┬──────┘           │
│         │                               │                   │
│         │    ┌──────────────────────────┘                   │
│         │    │                                              │
│         ▼    ▼                                              │
│  ┌───────────────────────────────────────────────┐         │
│  │                  Database                      │         │
│  │  ┌───────────────────┐ ┌───────────────────┐ │         │
│  │  │   Primary (Write) │ │  Replica (Read)   │ │         │
│  │  └───────────────────┘ └───────────────────┘ │         │
│  └───────────────────────────────────────────────┘         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                   Caching Strategy                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Read path (hot data caching):                              │
│                                                              │
│  GET /abc123                                                │
│     │                                                        │
│     ▼                                                        │
│  ┌─────────┐    cache hit     ┌─────────────┐              │
│  │  Cache  │─────────────────▶│  Redirect   │              │
│  │ (Redis) │                  └─────────────┘              │
│  └────┬────┘                                                │
│       │ cache miss                                          │
│       ▼                                                      │
│  ┌─────────┐                  ┌─────────────┐              │
│  │   DB    │─────────────────▶│ Update cache│              │
│  └─────────┘                  │ + Redirect  │              │
│                               └─────────────┘              │
│                                                              │
│  Cache design:                                               │
│  • Key: short_code                                          │
│  • Value: long_url                                          │
│  • TTL: 24 hours (popular URLs stay cached)                │
│  • Eviction: LRU                                            │
│                                                              │
│  Expected hit rate: 80%+ (Pareto: 20% URLs get 80% traffic)│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Trade-offs & Extensions

### 301 vs 302 Redirect

```
┌─────────────────────────────────────────────────────────────┐
│                301 vs 302 Redirect                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  301 (Permanent):                                           │
│  • Browser caches the redirect                             │
│  • Faster subsequent visits                                │
│  • Can't track clicks (browser doesn't hit server)        │
│                                                              │
│  302 (Temporary):                                           │
│  • Browser always hits server                              │
│  • Can track every click                                   │
│  • Slightly slower                                          │
│                                                              │
│  Choice: 302 if analytics needed, 301 for pure redirects  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Handling High Traffic

```
┌─────────────────────────────────────────────────────────────┐
│                Handling High Traffic                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Database sharding by short_code                        │
│     shard = hash(short_code) % num_shards                  │
│                                                              │
│  2. Multiple cache layers                                   │
│     CDN → Regional cache → Local cache → DB                │
│                                                              │
│  3. Separate read/write paths                              │
│     Writes: API servers → Primary DB                       │
│     Reads: API servers → Cache → Read replicas            │
│                                                              │
│  4. Rate limiting per user/IP                              │
│     Prevent abuse of shortening service                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Security Considerations

```
┌─────────────────────────────────────────────────────────────┐
│               Security Considerations                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Malicious URL prevention                               │
│     • Check against blocklist                              │
│     • Integrate with Google Safe Browsing API              │
│                                                              │
│  2. Non-predictable codes                                   │
│     • Don't use sequential IDs directly                    │
│     • Shuffle/randomize the encoding                       │
│                                                              │
│  3. Rate limiting                                           │
│     • Prevent bulk URL creation                            │
│     • Protect against enumeration attacks                  │
│                                                              │
│  4. Spam prevention                                         │
│     • CAPTCHA for anonymous users                          │
│     • Require authentication for high volume               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Key Generation** | Base62 with range allocation | No collisions, distributed |
| **Database** | NoSQL (Cassandra/DynamoDB) | Simple KV, horizontal scaling |
| **Cache** | Redis | Fast lookups, TTL support |
| **Redirect** | 302 | Enables analytics tracking |

---

[Back to System Designs](../README.md)
