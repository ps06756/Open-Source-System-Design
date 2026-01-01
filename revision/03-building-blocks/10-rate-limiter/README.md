# Rate Limiter

Rate limiting controls the rate of requests a client can make to prevent abuse and ensure fair usage.

## Table of Contents
- [Why Rate Limit?](#why-rate-limit)
- [Rate Limiting Algorithms](#rate-limiting-algorithms)
- [Distributed Rate Limiting](#distributed-rate-limiting)
- [Implementation](#implementation)
- [Key Takeaways](#key-takeaways)

## Why Rate Limit?

### Problems Without Rate Limiting

```
┌────────────────────────────────────────────────────────┐
│           Without Rate Limiting                        │
│                                                        │
│  Abusive client sends 10,000 req/sec                  │
│                     │                                  │
│                     ▼                                  │
│  ┌─────────────────────────────────────┐              │
│  │           Your Service              │ ← Overwhelmed│
│  └─────────────────────────────────────┘              │
│                     │                                  │
│  - Legitimate users can't access                      │
│  - Increased infrastructure costs                     │
│  - Potential cascading failures                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Prevent abuse | Stop malicious or buggy clients |
| Cost control | Limit resource consumption |
| Fair usage | Ensure all users get access |
| Stability | Prevent cascading failures |
| Security | Mitigate DDoS attacks |

## Rate Limiting Algorithms

### 1. Token Bucket

Tokens accumulate at a steady rate. Each request consumes a token.

```
┌────────────────────────────────────────────────────────┐
│                   Token Bucket                         │
│                                                        │
│  Bucket capacity: 10 tokens                           │
│  Refill rate: 1 token/second                          │
│                                                        │
│  ┌─────────────────────────────────────────┐          │
│  │ ● ● ● ● ● ● ● ○ ○ ○                    │◀── Tokens │
│  └─────────────────────────────────────────┘          │
│      7 tokens available                               │
│                                                        │
│  Request arrives:                                      │
│  - Token available? Remove token, allow request       │
│  - No token? Reject (429 Too Many Requests)          │
│                                                        │
│  Allows bursts up to bucket size                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:** Allows bursts, smooth average rate
**Cons:** Memory per user (store token count)

### 2. Leaky Bucket

Requests enter a bucket and "leak" out at a constant rate.

```
┌────────────────────────────────────────────────────────┐
│                   Leaky Bucket                         │
│                                                        │
│  Requests enter ────▶ ┌─────────┐                     │
│                       │ ● ● ● ● │ ← Queue             │
│                       │ ● ● ● ● │                     │
│                       └────┬────┘                     │
│                            │                          │
│                       Leak (constant rate)            │
│                            ▼                          │
│                      Process request                  │
│                                                        │
│  If bucket full: Reject new requests                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:** Smooth output rate, no bursts
**Cons:** May add latency (queuing)

### 3. Fixed Window Counter

Count requests in fixed time windows.

```
┌────────────────────────────────────────────────────────┐
│               Fixed Window Counter                     │
│                                                        │
│  Limit: 100 requests per minute                       │
│                                                        │
│  ├─────── Window 1 ────────┼─────── Window 2 ────────┤│
│  │     00:00 - 00:59       │     01:00 - 01:59       ││
│  │     Count: 87           │     Count: 23           ││
│  └─────────────────────────┴─────────────────────────┘│
│                                                        │
│  Problem: Boundary burst                              │
│  ├───────────────────┼───────────────────┤            │
│  │        90 req     │    90 req         │            │
│  │      (last 30s)   │  (first 30s)      │            │
│  └───────────────────┴───────────────────┘            │
│  180 requests in 1 minute! (exceeds 100 limit)       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:** Simple, memory efficient
**Cons:** Boundary burst problem

### 4. Sliding Window Log

Track timestamp of each request, count in sliding window.

```
┌────────────────────────────────────────────────────────┐
│               Sliding Window Log                       │
│                                                        │
│  Limit: 5 requests per 60 seconds                     │
│                                                        │
│  Request log:                                          │
│  [1:00:05, 1:00:15, 1:00:30, 1:00:45, 1:01:00]        │
│                                                        │
│  New request at 1:01:10:                              │
│  1. Remove entries older than 1:00:10                 │
│  2. Count remaining: 4                                │
│  3. Under limit? Allow and add timestamp              │
│                                                        │
│  Log: [1:00:15, 1:00:30, 1:00:45, 1:01:00, 1:01:10]  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:** Accurate, no boundary issues
**Cons:** Memory intensive (store all timestamps)

### 5. Sliding Window Counter

Combine fixed window with weighted calculation.

```
┌────────────────────────────────────────────────────────┐
│             Sliding Window Counter                     │
│                                                        │
│  Limit: 100 requests per minute                       │
│                                                        │
│  Previous window: 80 requests                         │
│  Current window: 30 requests                          │
│  Current position: 30 seconds into window (50%)      │
│                                                        │
│  Weighted count = prev × (1 - position) + current    │
│                 = 80 × 0.5 + 30                       │
│                 = 70 requests                          │
│                                                        │
│  70 < 100? Allow request                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:** Low memory, handles boundaries well
**Cons:** Approximation (not exact)

### Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst Handling |
|-----------|----------|--------|----------------|
| Token Bucket | Good | Medium | Allows bursts |
| Leaky Bucket | Good | Medium | Smooths bursts |
| Fixed Window | Low | Low | Boundary issue |
| Sliding Log | High | High | Accurate |
| Sliding Counter | Good | Low | Good approximation |

## Distributed Rate Limiting

### Challenge

```
┌────────────────────────────────────────────────────────┐
│           Distributed Challenge                        │
│                                                        │
│  User makes 100 requests:                             │
│  - 50 hit Server A (allows 50 of 100 limit)          │
│  - 50 hit Server B (allows 50 of 100 limit)          │
│  = 100 requests allowed (should be max 100!)          │
│                                                        │
│  ┌─────────────┐      ┌─────────────┐                 │
│  │  Server A   │      │  Server B   │                 │
│  │ Count: 50   │      │ Count: 50   │                 │
│  └─────────────┘      └─────────────┘                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Solution: Centralized Store

```
┌────────────────────────────────────────────────────────┐
│              Centralized Rate Limiting                 │
│                                                        │
│  ┌─────────────┐      ┌─────────────┐                 │
│  │  Server A   │──────│    Redis    │──────│ Server B ││
│  └─────────────┘      │  (counter)  │      └─────────┘│
│                       │  user:123   │                  │
│                       │  count: 100 │                  │
│                       └─────────────┘                  │
│                                                        │
│  All servers check/update the same counter            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Redis Implementation

```python
import redis
import time

def is_rate_limited(user_id, limit=100, window=60):
    """Sliding window counter using Redis"""
    r = redis.Redis()
    now = int(time.time())
    window_start = now - window
    key = f"rate:{user_id}"

    pipe = r.pipeline()

    # Remove old entries
    pipe.zremrangebyscore(key, 0, window_start)

    # Count requests in window
    pipe.zcard(key)

    # Add current request
    pipe.zadd(key, {str(now): now})

    # Set expiry
    pipe.expire(key, window)

    results = pipe.execute()
    request_count = results[1]

    return request_count >= limit
```

### Token Bucket with Redis

```lua
-- Redis Lua script for atomic token bucket
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Refill tokens
local elapsed = now - last_refill
local refill = elapsed * rate
tokens = math.min(capacity, tokens + refill)

-- Check and consume
if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, capacity / rate * 2)
    return 1  -- Allowed
else
    return 0  -- Denied
end
```

## Implementation

### Response Headers

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1640000000

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000030
```

### Rate Limiting Levels

```
┌────────────────────────────────────────────────────────┐
│             Rate Limiting Levels                       │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. Per-User                                           │
│     Key: user_id                                       │
│     Limit: 100 req/min                                │
│                                                        │
│  2. Per-IP                                             │
│     Key: client_ip                                     │
│     Limit: 1000 req/min                               │
│                                                        │
│  3. Per-Endpoint                                       │
│     Key: endpoint                                      │
│     Limit: 10000 req/min (global)                     │
│                                                        │
│  4. Per-User-Endpoint                                  │
│     Key: user_id + endpoint                           │
│     Limit: 10 req/min for expensive operations       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Tiered Limits

```yaml
# Different limits per plan
rate_limits:
  free:
    requests_per_minute: 60
    requests_per_day: 1000

  basic:
    requests_per_minute: 600
    requests_per_day: 50000

  enterprise:
    requests_per_minute: 6000
    requests_per_day: unlimited
```

## Key Takeaways

1. **Token bucket** is most flexible (allows bursts)
2. **Sliding window counter** is a good balance of accuracy/memory
3. **Use Redis** for distributed rate limiting
4. **Include headers** so clients know their limits
5. **Multiple levels** - user, IP, endpoint
6. **Graceful degradation** - 429 with Retry-After

## Practice Questions

1. Design a rate limiter for an API with 1M users.
2. How would you handle rate limiting across multiple data centers?
3. What's the trade-off between accuracy and performance?
4. How do you prevent a single user from impacting others?

## Further Reading

- [Stripe Rate Limiters](https://stripe.com/blog/rate-limiters)
- [Cloudflare Rate Limiting](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)

---

[Back to Module 3](../README.md) | Next: [Module 4: Design Patterns](../../04-design-patterns/README.md)
