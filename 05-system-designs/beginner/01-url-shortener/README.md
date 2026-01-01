# Design a URL Shortener (TinyURL)

Design a URL shortening service like TinyURL or bit.ly.

## 1. Requirements

### Functional Requirements
- Given a long URL, generate a short URL
- Given a short URL, redirect to the original URL
- Optional: Custom short URLs
- Optional: Analytics (click counts)
- Optional: Expiration

### Non-Functional Requirements
- High availability
- Low latency redirects (<100ms)
- Short URLs should be unpredictable
- Scale: 100M URLs created per month

### Capacity Estimation

```
Write: 100M URLs/month
     = 100M / (30 × 24 × 3600) ≈ 40 URLs/sec

Read (assume 100:1 read/write ratio):
     = 40 × 100 = 4,000 redirects/sec

Storage (5 years):
     = 100M × 12 × 5 = 6B URLs
     = 6B × 500 bytes = 3 TB

Bandwidth:
     Write: 40 × 500 bytes = 20 KB/s
     Read: 4,000 × 500 bytes = 2 MB/s
```

## 2. API Design

```
POST /api/shorten
Request:
{
    "long_url": "https://example.com/very/long/path",
    "custom_alias": "my-link",  // optional
    "expires_at": "2024-12-31"  // optional
}

Response:
{
    "short_url": "https://tiny.url/abc123",
    "long_url": "https://example.com/very/long/path",
    "expires_at": "2024-12-31",
    "created_at": "2024-01-15T10:30:00Z"
}

GET /{short_code}
Response: 301 Redirect to long_url

GET /api/stats/{short_code}
Response:
{
    "short_url": "https://tiny.url/abc123",
    "clicks": 15234,
    "created_at": "2024-01-15T10:30:00Z"
}
```

## 3. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Servers     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│                         ┌───────────────────────┼──────────┐  │
│                         │                       │          │  │
│                         ▼                       ▼          │  │
│                  ┌─────────────┐        ┌─────────────┐    │  │
│                  │    Cache    │        │  Database   │    │  │
│                  │   (Redis)   │        │ (PostgreSQL)│    │  │
│                  └─────────────┘        └─────────────┘    │  │
│                                                             │  │
│                                                             │  │
│  ┌──────────────────────────────────────────────────────┐  │  │
│  │                Key Generation Service                 │  │  │
│  │  (pre-generates unique keys for short URLs)          │  │  │
│  └──────────────────────────────────────────────────────┘  │  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 4. Short URL Generation

### Option 1: Base62 Encoding of Counter

```python
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

def encode_base62(num):
    if num == 0:
        return ALPHABET[0]

    result = []
    while num:
        result.append(ALPHABET[num % 62])
        num //= 62
    return ''.join(reversed(result))

# Examples:
# 1 → "1"
# 62 → "10"
# 1000000 → "4c92"
# 56800235584 → "1000000"
```

Length calculation:
- 6 chars: 62^6 = 56.8 billion URLs
- 7 chars: 62^7 = 3.5 trillion URLs

### Option 2: MD5/SHA256 Hash + Truncate

```python
import hashlib

def generate_short_code(long_url):
    hash_object = hashlib.md5(long_url.encode())
    hash_hex = hash_object.hexdigest()
    # Take first 7 characters
    return encode_base62(int(hash_hex[:12], 16))[:7]
```

**Problem**: Collisions possible

### Option 3: Pre-generated Keys (Recommended)

```
┌────────────────────────────────────────────────────────┐
│              Key Generation Service                    │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │           Key Database                            │ │
│  │  ┌────────────┬────────────┐                     │ │
│  │  │   key      │   used     │                     │ │
│  │  ├────────────┼────────────┤                     │ │
│  │  │  abc123    │   false    │                     │ │
│  │  │  xyz789    │   true     │                     │ │
│  │  │  ...       │   ...      │                     │ │
│  │  └────────────┴────────────┘                     │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  Pre-generate millions of unique keys                 │
│  Mark as used when assigned to a URL                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 5. Database Design

### Schema

```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id ON urls(user_id);

-- For analytics
CREATE TABLE clicks (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET,
    user_agent TEXT,
    referrer TEXT
);

CREATE INDEX idx_clicks_short_code ON clicks(short_code);
```

### SQL vs NoSQL

| Aspect | SQL (PostgreSQL) | NoSQL (DynamoDB) |
|--------|------------------|------------------|
| Query pattern | Simple key lookup | Simple key lookup |
| Scale | Moderate | Massive |
| Consistency | Strong | Eventual |
| Choice | Either works | Better for >1B URLs |

## 6. Detailed Flow

### Create Short URL

```
1. Client sends long URL
2. Check if URL already shortened (optional dedup)
3. Get unique key from Key Generation Service
4. Store mapping in database
5. Cache the mapping
6. Return short URL
```

### Redirect

```
1. Client requests short URL
2. Check cache for mapping
3. If cache miss, query database
4. If found, return 301 redirect
5. If not found, return 404
6. Async: increment click count
```

```python
def redirect(short_code):
    # Try cache first
    long_url = cache.get(short_code)

    if not long_url:
        # Cache miss - query database
        url_record = db.query(
            "SELECT long_url, expires_at FROM urls WHERE short_code = %s",
            short_code
        )

        if not url_record:
            return 404, "Not found"

        if url_record.expires_at and url_record.expires_at < now():
            return 410, "Gone (expired)"

        long_url = url_record.long_url
        cache.set(short_code, long_url, ttl=3600)

    # Async click tracking
    queue.publish("clicks", {
        "short_code": short_code,
        "timestamp": now(),
        "ip": request.ip
    })

    return 301, long_url
```

## 7. Scaling

### Database Scaling

```
┌────────────────────────────────────────────────────────┐
│                   Database Scaling                     │
│                                                        │
│  Option 1: Range-based sharding                       │
│  - Shard by first character of short_code            │
│  - a-j → Shard 1, k-t → Shard 2, u-z → Shard 3      │
│                                                        │
│  Option 2: Hash-based sharding                        │
│  - hash(short_code) % num_shards                     │
│  - Even distribution                                  │
│                                                        │
│  Option 3: Use DynamoDB/Cassandra                    │
│  - Built-in partitioning                             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Caching Strategy

```
┌────────────────────────────────────────────────────────┐
│                    Caching                             │
│                                                        │
│  What to cache: short_code → long_url                 │
│                                                        │
│  Cache size estimation:                               │
│  - 20% of URLs get 80% of traffic                    │
│  - Cache hot URLs: 6B × 0.2 = 1.2B entries           │
│  - Memory: 1.2B × 500 bytes = 600 GB                 │
│  - Use Redis Cluster                                  │
│                                                        │
│  TTL: 1 hour (refresh on access)                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 8. Additional Considerations

### Custom Aliases

```python
def create_custom_alias(long_url, custom_alias):
    # Validate alias (length, characters, not reserved)
    if not is_valid_alias(custom_alias):
        return error("Invalid alias")

    # Check availability
    if db.exists(custom_alias):
        return error("Alias already taken")

    # Reserve and create
    db.insert(custom_alias, long_url)
    return success(custom_alias)
```

### Analytics

```
Real-time: Redis counters
- INCR clicks:{short_code}

Historical: Batch processing
- Stream clicks to Kafka
- Process with Spark/Flink
- Store in data warehouse
```

### Rate Limiting

```
Per user: 100 URLs/hour
Per IP: 10 URLs/minute (anonymous)

Implementation: Token bucket in Redis
```

## 9. Trade-offs

| Decision | Trade-off |
|----------|-----------|
| 301 vs 302 redirect | 301 cacheable (faster) vs 302 trackable |
| Key length | Shorter (user-friendly) vs longer (more capacity) |
| Pre-generated keys | Extra infrastructure vs no collision handling |
| Sync vs async analytics | Accurate vs performant |

## 10. Key Takeaways

1. **Base62 encoding** is efficient for short URLs
2. **Pre-generated keys** avoid collision issues
3. **Cache heavily** - reads far exceed writes
4. **301 redirect** for SEO, 302 for analytics
5. **Shard by short_code** for horizontal scaling

---

Next: [Pastebin](../02-pastebin/README.md)
