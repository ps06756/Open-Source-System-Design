# Caching

Caching stores frequently accessed data in fast storage to reduce latency and backend load.

## Table of Contents
- [Why Cache?](#why-cache)
- [Cache Types](#cache-types)
- [Caching Patterns](#caching-patterns)
- [Cache Invalidation](#cache-invalidation)
- [Redis](#redis)
- [Key Takeaways](#key-takeaways)

## Why Cache?

```
Without cache:
Client → App → Database (100ms)

With cache:
Client → App → Cache (5ms) → 95% of requests
             → Database (100ms) → 5% cache misses

Average latency: 0.95 × 5ms + 0.05 × 100ms = 9.75ms
```

### Benefits

| Benefit | Impact |
|---------|--------|
| Lower latency | Milliseconds vs seconds |
| Reduced DB load | 10-100x fewer queries |
| Cost savings | Less infrastructure needed |
| Better UX | Faster page loads |

## Cache Types

### 1. Application Cache

```
┌────────────────────────────────────────────────────────┐
│                Application Cache                       │
│                                                        │
│  ┌─────────┐     ┌───────────┐     ┌──────────┐      │
│  │  App    │────▶│   Redis   │     │ Database │      │
│  │ Server  │◀────│  (cache)  │     │          │      │
│  └─────────┘     └───────────┘     └──────────┘      │
│       │                                   ▲           │
│       └───────────── cache miss ──────────┘           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 2. Database Cache

```
Built-in query cache:
MySQL: Query cache (deprecated in 8.0)
PostgreSQL: Buffer pool

┌────────────────────────────────────────────────────────┐
│                 PostgreSQL                             │
│  ┌─────────────────────────────────────────────┐      │
│  │              Shared Buffers                  │      │
│  │  (in-memory cache of disk pages)            │      │
│  └─────────────────────────────────────────────┘      │
│                         │                              │
│                         ▼                              │
│  ┌─────────────────────────────────────────────┐      │
│  │                  Disk                        │      │
│  └─────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
```

### 3. CDN Cache

See [CDN chapter](../03-cdn/README.md).

### 4. Browser Cache

```http
Cache-Control: max-age=86400
ETag: "abc123"
Last-Modified: Wed, 01 Jan 2024 00:00:00 GMT
```

## Caching Patterns

### Cache-Aside (Lazy Loading)

Application manages cache explicitly.

```
Read:
1. Check cache
2. If miss, read from DB
3. Store in cache
4. Return data

Write:
1. Write to DB
2. Invalidate cache

┌──────────────────────────────────────────┐
│  def get_user(user_id):                  │
│      # 1. Check cache                    │
│      user = cache.get(f"user:{user_id}") │
│      if user:                            │
│          return user                     │
│                                          │
│      # 2. Cache miss - read from DB      │
│      user = db.query(user_id)            │
│                                          │
│      # 3. Store in cache                 │
│      cache.set(f"user:{user_id}", user)  │
│                                          │
│      return user                         │
└──────────────────────────────────────────┘
```

**Pros:** Only caches what's needed
**Cons:** Cache miss penalty, potential stale data

### Read-Through

Cache handles DB reads transparently.

```
┌────────────────────────────────────────────────────────┐
│                   Read-Through                         │
│                                                        │
│  App ──▶ Cache ──(miss)──▶ Cache loads from DB        │
│      ◀────────────────────                            │
│                                                        │
│  Cache handles the DB query, not the app              │
└────────────────────────────────────────────────────────┘
```

**Pros:** Simpler application code
**Cons:** Requires cache provider support

### Write-Through

Write to cache and DB synchronously.

```
┌────────────────────────────────────────────────────────┐
│                  Write-Through                         │
│                                                        │
│  App ──▶ Cache ──▶ DB                                 │
│      ◀────────────────                                │
│                                                        │
│  Write completes when both succeed                    │
│                                                        │
│  Pros: Cache always consistent                        │
│  Cons: Higher write latency                           │
└────────────────────────────────────────────────────────┘
```

### Write-Behind (Write-Back)

Write to cache, async write to DB.

```
┌────────────────────────────────────────────────────────┐
│                   Write-Behind                         │
│                                                        │
│  App ──▶ Cache ──(async)──▶ DB                        │
│      ◀───(immediate)──                                │
│                                                        │
│  1. Write to cache (fast)                             │
│  2. Return success to app                             │
│  3. Background: flush to DB                           │
│                                                        │
│  Pros: Very fast writes                               │
│  Cons: Data loss risk if cache fails                 │
└────────────────────────────────────────────────────────┘
```

### Comparison

| Pattern | Read Latency | Write Latency | Consistency | Complexity |
|---------|--------------|---------------|-------------|------------|
| Cache-Aside | Miss penalty | Low | Eventual | Medium |
| Read-Through | Miss penalty | Low | Eventual | Low |
| Write-Through | Low | High | Strong | Medium |
| Write-Behind | Low | Very Low | Eventual | High |

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

### Strategies

**1. TTL (Time-To-Live)**
```python
cache.set("user:123", user_data, ttl=3600)  # Expires in 1 hour
```

**2. Event-Based Invalidation**
```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")  # Invalidate on write
```

**3. Version-Based**
```python
version = 5
cache.set(f"user:{user_id}:v{version}", user_data)
# Change version to invalidate
```

### Cache Stampede Prevention

```
Problem:
Popular key expires → 1000 requests hit DB simultaneously

Solutions:

1. Lock/Mutex
   First request acquires lock, others wait
   cache.lock("user:123")

2. Early Expiration
   Refresh before actual expiry
   if ttl < threshold:
       refresh_async()

3. Probabilistic Early Expiration
   Random refresh before expiry
```

## Redis

The most popular in-memory cache and data structure store.

### Data Structures

```redis
# Strings
SET user:123:name "John"
GET user:123:name

# Hashes
HSET user:123 name "John" email "john@example.com"
HGETALL user:123

# Lists (queues)
LPUSH queue:tasks "task1"
RPOP queue:tasks

# Sets (unique values)
SADD user:123:followers "user:456"
SISMEMBER user:123:followers "user:456"

# Sorted Sets (leaderboards)
ZADD leaderboard 100 "user:123"
ZRANGE leaderboard 0 9 WITHSCORES

# Pub/Sub
PUBLISH channel:updates "new message"
SUBSCRIBE channel:updates
```

### Redis Patterns

**Session Storage**
```python
# Store session
redis.setex(f"session:{session_id}", 3600, json.dumps(session_data))

# Get session
session = json.loads(redis.get(f"session:{session_id}"))
```

**Rate Limiting**
```python
def is_rate_limited(user_id):
    key = f"rate:{user_id}:{current_minute()}"
    count = redis.incr(key)
    redis.expire(key, 60)
    return count > 100
```

**Distributed Lock**
```python
def acquire_lock(resource, timeout=10):
    return redis.set(f"lock:{resource}", "1",
                     nx=True, ex=timeout)

def release_lock(resource):
    redis.delete(f"lock:{resource}")
```

### Redis Cluster

```
┌────────────────────────────────────────────────────────┐
│                   Redis Cluster                        │
│                                                        │
│  16384 hash slots distributed across nodes            │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
│  │   Node 1     │  │   Node 2     │  │   Node 3     ││
│  │ Slots 0-5460 │  │ Slots 5461-  │  │ Slots 10923- ││
│  │              │  │     10922    │  │     16383    ││
│  └──────────────┘  └──────────────┘  └──────────────┘│
│         │                 │                 │         │
│         ▼                 ▼                 ▼         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
│  │   Replica    │  │   Replica    │  │   Replica    ││
│  └──────────────┘  └──────────────┘  └──────────────┘│
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | Rich | Strings only |
| Persistence | Yes | No |
| Replication | Yes | No |
| Clustering | Yes | Client-side |
| Lua scripting | Yes | No |
| Memory efficiency | Lower | Higher |
| Use case | Feature-rich | Simple caching |

## Key Takeaways

1. **Cache-aside** is most common and flexible
2. **TTL is simple** but may serve stale data
3. **Invalidation is hard** - design for it upfront
4. **Redis** is versatile (cache, queue, pub/sub)
5. **Cache hit ratio** should be >90% for effectiveness
6. **Cache stampede** can bring down your database

## Practice Questions

1. Design a caching strategy for a news feed.
2. How would you handle cache invalidation for related data?
3. Calculate the cache size needed for 1M users.
4. When would you choose Memcached over Redis?

## Further Reading

- [Redis Documentation](https://redis.io/documentation)
- [Caching Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/BestPractices.html)
- [Facebook's Memcached Paper](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)

---

Next: [Message Queues](../06-message-queues/README.md)
