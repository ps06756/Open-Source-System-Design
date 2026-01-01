# Design Pastebin

## 1. Requirements

### Functional Requirements
- Users can paste text and get a unique URL
- Users can access paste via URL
- Optional: Syntax highlighting
- Optional: Expiration time
- Optional: Password protection
- Optional: User accounts with paste management

### Non-Functional Requirements
- High availability
- Low latency for reads
- Content size limit (e.g., 10 MB per paste)
- Durability (don't lose pastes)

### Scale Estimates
- 1M new pastes per day
- 10:1 read to write ratio
- Average paste size: 10 KB
- Paste retention: 1 year

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Writes: 1M/day = ~12 pastes/second                        │
│  Reads: 10M/day = ~120 reads/second                        │
│  Storage: 1M × 365 × 10KB = 3.6 TB/year                   │
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
│  Create Paste:                                               │
│  ┌────────┐   POST /paste    ┌─────────────┐               │
│  │ Client │─────────────────▶│ API Server  │               │
│  └────────┘                  └──────┬──────┘               │
│                                     │                       │
│                    ┌────────────────┴────────────────┐      │
│                    ▼                                 ▼      │
│             ┌───────────┐                    ┌───────────┐ │
│             │  Key Gen  │                    │   Blob    │ │
│             │  Service  │                    │  Storage  │ │
│             └───────────┘                    └───────────┘ │
│                    │                                        │
│                    ▼                                        │
│             ┌───────────┐                                   │
│             │  Metadata │                                   │
│             │    DB     │                                   │
│             └───────────┘                                   │
│                                                              │
│  Read Paste:                                                 │
│  ┌────────┐   GET /{key}     ┌───────────┐   ┌──────────┐  │
│  │ Client │─────────────────▶│ API Server│──▶│  Cache   │  │
│  └────────┘                  └─────┬─────┘   └────┬─────┘  │
│       ▲                           │     Miss     │         │
│       │                           │              ▼         │
│       │                           │        ┌──────────┐    │
│       └───────────────────────────┘        │  Blob    │    │
│              Return content               │  Storage │    │
│                                            └──────────┘    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### API Design

```
POST /api/v1/paste
Request:
{
    "content": "Hello, World!",
    "expiration": "1d",  // 1h, 1d, 1w, 1m, never
    "syntax": "python",  // optional
    "password": "secret" // optional
}
Response:
{
    "key": "abc123",
    "url": "https://paste.io/abc123",
    "expires_at": "2024-01-16T00:00:00Z"
}

GET /api/v1/paste/{key}
Query: ?password=secret  // if password protected
Response:
{
    "content": "Hello, World!",
    "syntax": "python",
    "created_at": "2024-01-15T00:00:00Z",
    "expires_at": "2024-01-16T00:00:00Z"
}
```

---

## 3. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Metadata Database (SQL):                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Table: pastes                                           │ │
│  │ ┌────────────────┬──────────────────────────────────┐  │ │
│  │ │ key (PK)       │ VARCHAR(8)                        │  │ │
│  │ │ user_id        │ BIGINT (nullable)                 │  │ │
│  │ │ content_path   │ VARCHAR(255) - S3 path           │  │ │
│  │ │ content_size   │ INT                               │  │ │
│  │ │ syntax         │ VARCHAR(50)                       │  │ │
│  │ │ password_hash  │ VARCHAR(255) (nullable)           │  │ │
│  │ │ created_at     │ TIMESTAMP                         │  │ │
│  │ │ expires_at     │ TIMESTAMP (nullable)              │  │ │
│  │ │ view_count     │ INT DEFAULT 0                     │  │ │
│  │ └────────────────┴──────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Blob Storage (S3):                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Bucket: pastebin-content                                │ │
│  │ Path: /pastes/{year}/{month}/{day}/{key}               │ │
│  │ Content: Raw paste text                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Why separate metadata and content?                        │
│  • Metadata is small, queried frequently → SQL            │
│  • Content is large, read-only → Blob storage             │
│  • Different scaling characteristics                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Key Generation

```
┌─────────────────────────────────────────────────────────────┐
│                   Key Generation                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Same approaches as URL Shortener:                          │
│                                                              │
│  1. Pre-generated keys (recommended)                       │
│     ┌──────────────────────────────────────────────────────┐│
│     │  Key Generation Service                              ││
│     │  ┌────────────┐                                      ││
│     │  │ Unused Keys│ ──────▶ API Server ──────▶ Used Keys ││
│     │  │   Pool     │  (take one)       (save mapping)    ││
│     │  └────────────┘                                      ││
│     │                                                       ││
│     │  Background job continuously generates new keys      ││
│     └──────────────────────────────────────────────────────┘│
│                                                              │
│  2. Counter with Base62                                     │
│     Same as URL shortener                                  │
│                                                              │
│  Key length: 8 characters                                  │
│  62^8 = 218 trillion combinations (more than enough!)      │
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
│                    ┌─────────────┐                          │
│                    │     CDN     │                          │
│                    └──────┬──────┘                          │
│                           │                                  │
│                    ┌──────┴──────┐                          │
│                    │ Load Balancer│                         │
│                    └──────┬──────┘                          │
│           ┌───────────────┼───────────────┐                 │
│           ▼               ▼               ▼                 │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ API Server  │ │ API Server  │ │ API Server  │           │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘           │
│         │               │               │                   │
│         └───────────────┼───────────────┘                   │
│                         │                                    │
│  ┌──────────────────────┼──────────────────────┐            │
│  │                      │                      │            │
│  ▼                      ▼                      ▼            │
│ ┌──────────┐     ┌───────────┐          ┌───────────┐      │
│ │  Redis   │     │ Metadata  │          │   S3      │      │
│ │  Cache   │     │   DB      │          │  Bucket   │      │
│ └──────────┘     │ (Primary) │          └───────────┘      │
│                  └─────┬─────┘                              │
│                        │                                    │
│                        ▼                                    │
│                  ┌───────────┐                              │
│                  │  Replica  │                              │
│                  └───────────┘                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Background Services                                  │  │
│  │  ┌──────────────┐  ┌──────────────┐                  │  │
│  │  │ Key Gen Job  │  │ Cleanup Job  │                  │  │
│  │  │ (pre-generate│  │ (delete      │                  │  │
│  │  │  keys)       │  │  expired)    │                  │  │
│  │  └──────────────┘  └──────────────┘                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Write Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      Write Flow                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Client sends POST /paste with content                  │
│                                                              │
│  2. API Server:                                             │
│     a. Validate content (size, format)                     │
│     b. Get unique key from Key Generation Service          │
│     c. Upload content to S3 (async if large)              │
│     d. Store metadata in database                          │
│     e. Return key to client                                │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def create_paste(content, options):                   │ │
│  │      # Validate                                         │ │
│  │      if len(content) > MAX_SIZE:                       │ │
│  │          raise ContentTooLarge()                       │ │
│  │                                                         │ │
│  │      # Get key                                          │ │
│  │      key = key_service.get_unique_key()                │ │
│  │                                                         │ │
│  │      # Upload to S3                                     │ │
│  │      path = f"pastes/{date}/{key}"                     │ │
│  │      s3.upload(path, content)                          │ │
│  │                                                         │ │
│  │      # Store metadata                                   │ │
│  │      db.insert({                                        │ │
│  │          key: key,                                      │ │
│  │          content_path: path,                           │ │
│  │          expires_at: calculate_expiry(options),        │ │
│  │          ...                                            │ │
│  │      })                                                 │ │
│  │                                                         │ │
│  │      return key                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Read Flow

```
┌─────────────────────────────────────────────────────────────┐
│                       Read Flow                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Check cache for content                                 │
│  2. If miss, get metadata from DB                          │
│  3. If expired/not found, return 404                       │
│  4. If password protected, verify password                 │
│  5. Fetch content from S3                                  │
│  6. Cache the content                                      │
│  7. Return to client                                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def get_paste(key, password=None):                    │ │
│  │      # Check cache                                      │ │
│  │      cached = cache.get(key)                           │ │
│  │      if cached:                                         │ │
│  │          return cached                                  │ │
│  │                                                         │ │
│  │      # Get metadata                                     │ │
│  │      metadata = db.get(key)                            │ │
│  │      if not metadata:                                   │ │
│  │          raise NotFound()                              │ │
│  │                                                         │ │
│  │      # Check expiration                                 │ │
│  │      if metadata.expires_at < now():                   │ │
│  │          raise NotFound()                              │ │
│  │                                                         │ │
│  │      # Check password                                   │ │
│  │      if metadata.password_hash:                        │ │
│  │          if not verify(password, metadata.password):   │ │
│  │              raise Unauthorized()                      │ │
│  │                                                         │ │
│  │      # Fetch content                                    │ │
│  │      content = s3.get(metadata.content_path)           │ │
│  │                                                         │ │
│  │      # Cache and return                                 │ │
│  │      cache.set(key, content, ttl=1h)                   │ │
│  │      return content                                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                   Caching Strategy                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Multi-layer caching:                                       │
│                                                              │
│  1. CDN (edge caching)                                      │
│     • Cache static assets                                   │
│     • Cache public pastes at edge                          │
│     • TTL based on popularity                              │
│                                                              │
│  2. Application cache (Redis)                              │
│     • Recent pastes                                         │
│     • Hot pastes (frequently accessed)                     │
│     • TTL: 1 hour                                          │
│                                                              │
│  3. S3 (origin)                                             │
│     • All pastes                                            │
│     • Durable storage                                       │
│                                                              │
│  Cache key design:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Key: paste:{key}                                       │ │
│  │  Value: {                                               │ │
│  │    "content": "...",                                    │ │
│  │    "syntax": "python",                                  │ │
│  │    "expires_at": "..."                                  │ │
│  │  }                                                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Eviction: LRU + TTL                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Cleanup Service

```
┌─────────────────────────────────────────────────────────────┐
│                   Cleanup Service                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Cron job runs periodically to clean expired pastes:       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def cleanup_expired_pastes():                          │ │
│  │      # Find expired pastes                              │ │
│  │      expired = db.query(                               │ │
│  │          "SELECT key, content_path FROM pastes         │ │
│  │           WHERE expires_at < NOW()                     │ │
│  │           LIMIT 1000"                                  │ │
│  │      )                                                  │ │
│  │                                                         │ │
│  │      for paste in expired:                             │ │
│  │          # Delete from S3                              │ │
│  │          s3.delete(paste.content_path)                 │ │
│  │                                                         │ │
│  │          # Delete from cache                           │ │
│  │          cache.delete(f"paste:{paste.key}")            │ │
│  │                                                         │ │
│  │          # Delete from DB                              │ │
│  │          db.delete(paste.key)                          │ │
│  │                                                         │ │
│  │      log(f"Cleaned {len(expired)} pastes")             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Schedule: Every hour                                       │
│  Batch size: 1000 (to avoid long transactions)             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Trade-offs & Extensions

### Small vs Large Content

```
┌─────────────────────────────────────────────────────────────┐
│              Small vs Large Content                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Optimization: Store small content inline                  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  If content < 1KB:                                      │ │
│  │    Store directly in metadata DB                       │ │
│  │    (avoids S3 roundtrip)                               │ │
│  │                                                         │ │
│  │  If content >= 1KB:                                     │ │
│  │    Store in S3                                          │ │
│  │    (better for large files)                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Table modification:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ content_inline  │ TEXT (nullable) - for small pastes   │ │
│  │ content_path    │ VARCHAR (nullable) - for large pastes│ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Rate Limiting

```
┌─────────────────────────────────────────────────────────────┐
│                    Rate Limiting                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Prevent abuse:                                             │
│  • Anonymous: 10 pastes/hour/IP                            │
│  • Registered: 100 pastes/hour/user                        │
│  • Premium: 1000 pastes/hour/user                          │
│                                                              │
│  Implementation: Token bucket with Redis                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Metadata Storage** | PostgreSQL | Structured queries, transactions |
| **Content Storage** | S3 | Cheap, durable, scalable |
| **Cache** | Redis | Fast reads, TTL support |
| **Key Generation** | Pre-generated pool | Fast, no collisions |
| **CDN** | CloudFront | Edge caching, low latency |

---

[Back to System Designs](../README.md)
