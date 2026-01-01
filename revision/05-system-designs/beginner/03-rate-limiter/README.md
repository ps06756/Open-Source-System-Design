# Design a Rate Limiter

Design a distributed rate limiting system to protect APIs from abuse.

## 1. Requirements

### Functional Requirements
- Limit requests per user/IP/API key
- Support multiple rate limiting rules
- Return appropriate headers (remaining, reset time)
- Low latency (add <10ms to request)

### Non-Functional Requirements
- Highly available
- Low latency
- Distributed (work across multiple servers)
- Accurate (no significant over/under counting)

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Rate Limiter │────▶│    API Servers     │   │
│  └────────┘     │  (Middleware)│     └────────────────────┘   │
│                 └──────┬───────┘                              │
│                        │                                       │
│                        ▼                                       │
│                 ┌─────────────┐                               │
│                 │    Redis    │                               │
│                 │  (Counters) │                               │
│                 └─────────────┘                               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Rate Limiting Algorithms

### Token Bucket

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity      # Max tokens
        self.refill_rate = refill_rate  # Tokens per second

    def allow_request(self, key, tokens=1):
        now = time.time()
        bucket = self.get_bucket(key)

        # Refill tokens
        elapsed = now - bucket.last_refill
        bucket.tokens = min(
            self.capacity,
            bucket.tokens + elapsed * self.refill_rate
        )
        bucket.last_refill = now

        # Check and consume
        if bucket.tokens >= tokens:
            bucket.tokens -= tokens
            self.save_bucket(key, bucket)
            return True
        return False
```

### Sliding Window Counter (Redis)

```python
def is_rate_limited(user_id, limit=100, window=60):
    """
    Sliding window using Redis sorted set
    """
    key = f"rate:{user_id}"
    now = time.time()
    window_start = now - window

    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)  # Remove old
    pipe.zcard(key)                               # Count
    pipe.zadd(key, {str(now): now})              # Add current
    pipe.expire(key, window)                      # Set TTL

    results = pipe.execute()
    count = results[1]

    return count >= limit
```

### Token Bucket with Redis Lua

```lua
-- Atomic token bucket in Redis
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'timestamp')
local tokens = tonumber(data[1]) or capacity
local last_time = tonumber(data[2]) or now

-- Refill
local elapsed = now - last_time
tokens = math.min(capacity, tokens + elapsed * rate)

-- Check
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'timestamp', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)
    return {1, tokens}  -- Allowed
else
    return {0, tokens}  -- Denied
end
```

## 4. Distributed Rate Limiting

```
┌────────────────────────────────────────────────────────┐
│           Distributed Challenges                       │
│                                                        │
│  Problem: User hits different servers                 │
│                                                        │
│  Server A: count = 50 (of 100 limit)                 │
│  Server B: count = 50 (of 100 limit)                 │
│  Total: 100 requests allowed!                         │
│                                                        │
│  Solution: Centralized counter (Redis)               │
│                                                        │
│  ┌─────────┐      ┌─────────┐                        │
│  │Server A │─────▶│  Redis  │◀──────│Server B │      │
│  └─────────┘      │(central)│       └─────────┘      │
│                   └─────────┘                         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Multi-Region Considerations

```
Option 1: Regional limits
- Each region has own limit
- User can get 100 req in US + 100 in EU
- Simple but allows gaming

Option 2: Global Redis
- Single source of truth
- Higher latency for distant regions

Option 3: Eventual sync
- Local counters
- Periodic sync to global
- May over-allow during sync window
```

## 5. API Response

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1640000000

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
Retry-After: 30
```

## 6. Configuration

```yaml
rate_limits:
  - name: api_default
    key: user_id
    limit: 100
    window: 60  # seconds

  - name: api_sensitive
    path: /api/payments
    key: user_id
    limit: 10
    window: 60

  - name: anonymous
    key: ip_address
    limit: 20
    window: 60
```

## 7. Key Takeaways

1. **Token bucket** is most flexible (allows bursts)
2. **Sliding window counter** is good balance
3. **Use Redis** for distributed rate limiting
4. **Lua scripts** for atomic operations
5. **Return headers** so clients can adapt
6. **Graceful degradation** when Redis is down

---

Next: [Key-Value Store](../04-key-value-store/README.md)
