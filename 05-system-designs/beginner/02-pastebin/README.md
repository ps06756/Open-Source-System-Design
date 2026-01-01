# Design Pastebin

Design a text sharing service like Pastebin where users can store and share plain text.

## 1. Requirements

### Functional Requirements
- Create paste with text content
- Get paste by unique URL
- Optional: Expiration time
- Optional: Private/public pastes
- Optional: Syntax highlighting

### Non-Functional Requirements
- High availability
- Low latency reads (<100ms)
- Paste data is immutable
- Scale: 10M new pastes/month

### Capacity Estimation

```
Writes: 10M pastes/month ≈ 4 pastes/sec
Reads (10:1 ratio): 40 reads/sec

Storage (5 years):
- 10M × 12 × 5 = 600M pastes
- Average paste: 10 KB
- Total: 600M × 10 KB = 6 TB

Metadata: 600M × 100 bytes = 60 GB
```

## 2. API Design

```
POST /api/paste
Request:
{
    "content": "print('Hello, World!')",
    "title": "My Python Script",
    "syntax": "python",
    "expires_in": 86400,  // seconds
    "is_private": false
}

Response:
{
    "paste_id": "abc123xyz",
    "url": "https://pastebin.com/abc123xyz",
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2024-01-16T10:30:00Z"
}

GET /api/paste/{paste_id}
Response:
{
    "paste_id": "abc123xyz",
    "content": "print('Hello, World!')",
    "title": "My Python Script",
    "syntax": "python",
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
│                    ┌────────────────────────────┼───────────┐ │
│                    │                            │           │ │
│                    ▼                            ▼           │ │
│             ┌─────────────┐            ┌─────────────────┐  │ │
│             │    Cache    │            │   Metadata DB   │  │ │
│             │   (Redis)   │            │   (PostgreSQL)  │  │ │
│             └─────────────┘            └─────────────────┘  │ │
│                                                 │           │ │
│                                                 ▼           │ │
│                                        ┌─────────────────┐  │ │
│                                        │  Object Storage │  │ │
│                                        │      (S3)       │  │ │
│                                        └─────────────────┘  │ │
│                                                              │ │
└────────────────────────────────────────────────────────────────┘
```

## 4. Data Storage

### Separation of Concerns

```
Metadata (PostgreSQL):
- paste_id, title, syntax, created_at, expires_at, user_id

Content (S3):
- Actual text content
- Cheaper storage for large text
- Easy to scale
```

### Database Schema

```sql
CREATE TABLE pastes (
    paste_id VARCHAR(12) PRIMARY KEY,
    title VARCHAR(255),
    syntax VARCHAR(50),
    content_url TEXT NOT NULL,  -- S3 URL
    content_size INTEGER,
    user_id BIGINT,
    is_private BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    view_count BIGINT DEFAULT 0
);

CREATE INDEX idx_user_id ON pastes(user_id);
CREATE INDEX idx_expires_at ON pastes(expires_at);
```

### Content Storage

```python
def create_paste(content, metadata):
    paste_id = generate_unique_id()  # Base62 encoded

    # Store content in S3
    s3_key = f"pastes/{paste_id[:2]}/{paste_id}"
    s3.put_object(
        Bucket="pastebin-content",
        Key=s3_key,
        Body=content.encode('utf-8'),
        ContentType="text/plain"
    )

    # Store metadata in PostgreSQL
    db.insert("pastes", {
        "paste_id": paste_id,
        "content_url": s3_key,
        "content_size": len(content),
        **metadata
    })

    return paste_id
```

## 5. Expiration Handling

```
┌────────────────────────────────────────────────────────┐
│              Cleanup Service                           │
│                                                        │
│  Option 1: Lazy deletion                              │
│  - Check expiration on read                           │
│  - Return 404 if expired                              │
│  - Background job cleans up                           │
│                                                        │
│  Option 2: Active deletion                            │
│  - Cron job runs every hour                           │
│  - DELETE FROM pastes WHERE expires_at < NOW()        │
│  - Delete S3 objects                                  │
│                                                        │
│  Recommendation: Lazy + Active combined               │
│                                                        │
└────────────────────────────────────────────────────────┘

def get_paste(paste_id):
    paste = cache.get(paste_id) or db.get(paste_id)

    if not paste:
        return 404

    if paste.expires_at and paste.expires_at < now():
        # Lazy deletion
        return 410  # Gone

    return paste
```

## 6. Caching Strategy

```
┌────────────────────────────────────────────────────────┐
│                  Caching                               │
│                                                        │
│  Cache metadata + content for hot pastes              │
│                                                        │
│  Key: paste:{paste_id}                                │
│  Value: { metadata, content }                         │
│  TTL: 1 hour                                          │
│                                                        │
│  Estimation:                                           │
│  - 20% pastes = 80% reads                            │
│  - Hot pastes: 600M × 0.2 = 120M                     │
│  - Memory: 120M × 10KB = 1.2 TB                      │
│  - Use Redis Cluster                                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 7. Scaling

### Read Path Optimization

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Client ──▶ CDN ──▶ API Server ──▶ Cache ──▶ Storage  │
│              │                                          │
│              └── Cache popular pastes at CDN edge      │
│                                                         │
│  For public pastes:                                     │
│  - CDN caches content                                  │
│  - Cache-Control: public, max-age=3600                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Database Sharding

```
Shard by paste_id (hash-based):
- Shard 1: hash(paste_id) % 4 == 0
- Shard 2: hash(paste_id) % 4 == 1
- Shard 3: hash(paste_id) % 4 == 2
- Shard 4: hash(paste_id) % 4 == 3
```

## 8. Key Takeaways

1. **Separate metadata from content** - different storage needs
2. **Use object storage (S3)** for content - cheap and scalable
3. **Lazy + active deletion** for expiration
4. **CDN for public pastes** - reduces load significantly
5. **Similar to URL shortener** but with larger payloads

---

Next: [Rate Limiter](../03-rate-limiter/README.md)
