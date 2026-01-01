# Design a Rate Limiter

## 1. Requirements

### Functional Requirements
- Limit requests per client based on configured rules
- Support multiple limit types (per user, IP, API key)
- Return appropriate response when rate limited (429)
- Provide rate limit headers to clients

### Non-Functional Requirements
- Low latency (< 1ms overhead)
- Highly available
- Accurate (don't allow more than limit)
- Distributed (works across multiple servers)

### Rate Limit Rules Examples
```
- User API: 1000 requests/hour/user
- Login endpoint: 5 requests/minute/IP
- Public API: 100 requests/day/API key
- Search endpoint: 20 requests/minute/user
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Request Flow:                                               │
│                                                              │
│  ┌────────┐    ┌───────────────┐    ┌─────────────────────┐ │
│  │ Client │───▶│ Rate Limiter  │───▶│   API Server        │ │
│  └────────┘    │ (Middleware)  │    │                     │ │
│       ▲        └───────┬───────┘    └─────────────────────┘ │
│       │                │                                     │
│       │                │ Check rate                          │
│       │                ▼                                     │
│       │        ┌───────────────┐                            │
│       │        │  Rate Limit   │                            │
│       │        │    Store      │                            │
│       │        │   (Redis)     │                            │
│       │        └───────────────┘                            │
│       │                │                                     │
│       │    429 Too Many│Requests                            │
│       └────────────────┘                                    │
│                                                              │
│  Where to place Rate Limiter?                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Option 1: API Gateway (recommended for simplicity)    │ │
│  │  Option 2: Middleware in each service                  │ │
│  │  Option 3: Separate Rate Limiter service              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Rate Limiting Algorithms

### Token Bucket (Recommended)

```
┌─────────────────────────────────────────────────────────────┐
│                      Token Bucket                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Parameters:                                                 │
│  • Bucket capacity: 10 tokens                               │
│  • Refill rate: 10 tokens/minute                           │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Time  │ Tokens │ Requests │ Result                    │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │ 0:00  │   10   │     5    │ Allowed (5 remaining)     │  │
│  │ 0:15  │   5+2  │     8    │ Allowed (0 remaining)     │  │
│  │ 0:20  │   0+1  │     2    │ 1 allowed, 1 rejected     │  │
│  │ 0:30  │   1+2  │     0    │ 3 tokens saved            │  │
│  │ 1:00  │   3+5  │    10    │ 8 allowed, 2 rejected     │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  Pros:                                                       │
│  ✓ Allows controlled bursts                                 │
│  ✓ Memory efficient (2 values per key)                     │
│  ✓ Easy to implement                                        │
│                                                              │
│  Cons:                                                       │
│  ✗ Parameters need tuning                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Redis Implementation:**
```lua
-- Token Bucket Lua Script (atomic execution)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

-- Calculate tokens to add
local elapsed = now - last_refill
local tokens_to_add = elapsed * rate
tokens = math.min(capacity, tokens + tokens_to_add)

-- Check if request can be allowed
local allowed = tokens >= requested
if allowed then
    tokens = tokens - requested
end

-- Update bucket
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, capacity / rate * 2)  -- TTL for cleanup

return allowed and 1 or 0
```

### Sliding Window Counter

```
┌─────────────────────────────────────────────────────────────┐
│                Sliding Window Counter                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Limit: 100 requests per minute                            │
│                                                              │
│  ┌─────────────────────┬─────────────────────┐             │
│  │   Previous Window   │   Current Window    │             │
│  │   (0:00 - 1:00)     │   (1:00 - 2:00)     │             │
│  │   Count: 80         │   Count: 30         │             │
│  └─────────────────────┴─────────────────────┘             │
│                        ↑                                    │
│              Current time: 1:15 (25% into window)          │
│                                                              │
│  Weighted count = prev × (1 - elapsed%) + curr             │
│                 = 80 × 0.75 + 30                           │
│                 = 60 + 30 = 90                             │
│                                                              │
│  90 < 100 → Allow request                                  │
│                                                              │
│  Pros:                                                       │
│  ✓ Smooth rate limiting                                    │
│  ✓ Memory efficient (2 counters)                           │
│  ✓ No burst at window boundaries                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Redis Implementation:**
```lua
-- Sliding Window Counter Lua Script
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])  -- window size in seconds
local now = tonumber(ARGV[3])

local current_window = math.floor(now / window) * window
local previous_window = current_window - window

local current_key = key .. ':' .. current_window
local previous_key = key .. ':' .. previous_window

local current_count = tonumber(redis.call('GET', current_key) or 0)
local previous_count = tonumber(redis.call('GET', previous_key) or 0)

-- Calculate weighted count
local elapsed = now - current_window
local weight = 1 - (elapsed / window)
local weighted_count = previous_count * weight + current_count

if weighted_count >= limit then
    return 0  -- Rejected
end

-- Increment current window
redis.call('INCR', current_key)
redis.call('EXPIRE', current_key, window * 2)

return 1  -- Allowed
```

### Fixed Window Counter

```
┌─────────────────────────────────────────────────────────────┐
│                 Fixed Window Counter                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Simplest approach but has boundary burst problem:         │
│                                                              │
│  ┌─────────────────────┬─────────────────────┐             │
│  │   Window 0:00-1:00  │   Window 1:00-2:00  │             │
│  │                     │                     │             │
│  │         ↓ 100 here  │ 100 here ↓         │             │
│  │    ....●●●●●●●●●●●●│●●●●●●●●●●●●....    │             │
│  │       0:55         │  1:05              │             │
│  └─────────────────────┴─────────────────────┘             │
│                                                              │
│  200 requests in 10 seconds! (limit is 100/minute)         │
│                                                              │
│  Redis Implementation:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  window_key = f"ratelimit:{user}:{minute}"             │ │
│  │  count = redis.incr(window_key)                        │ │
│  │  if count == 1:                                         │ │
│  │      redis.expire(window_key, 60)                      │ │
│  │  if count > limit:                                      │ │
│  │      return RATE_LIMITED                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Pros: Simple, memory efficient                            │
│  Cons: Burst at window boundaries                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Leaky Bucket

```
┌─────────────────────────────────────────────────────────────┐
│                      Leaky Bucket                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Smooths burst traffic to constant rate output             │
│                                                              │
│       Incoming requests (bursty)                            │
│            ↓ ↓ ↓ ↓ ↓ ↓                                      │
│       ┌────────────────┐                                    │
│       │     Queue      │  ← If full, drop requests         │
│       │  [r][r][r][r]  │                                    │
│       └───────┬────────┘                                    │
│               │                                              │
│               ▼ constant rate                               │
│          ┌─────────┐                                        │
│          │ Process │                                        │
│          └─────────┘                                        │
│                                                              │
│  Implementation: Process queue at fixed rate               │
│                                                              │
│  Use case: Traffic shaping, where smooth output matters    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Distributed Rate Limiting

```
┌─────────────────────────────────────────────────────────────┐
│              Distributed Rate Limiting                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Challenge: Multiple servers, consistent rate limiting     │
│                                                              │
│  ┌────────┐     ┌──────────┐                               │
│  │ Server │────▶│          │                               │
│  │   1    │     │          │                               │
│  └────────┘     │  Redis   │                               │
│  ┌────────┐     │ Cluster  │                               │
│  │ Server │────▶│          │                               │
│  │   2    │     │          │                               │
│  └────────┘     └──────────┘                               │
│  ┌────────┐          │                                      │
│  │ Server │──────────┘                                      │
│  │   3    │                                                 │
│  └────────┘                                                 │
│                                                              │
│  Race Condition Solution:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Use Lua scripts for atomic operations                 │ │
│  │  - Read, check, and update in single operation         │ │
│  │  - Redis executes Lua atomically                       │ │
│  │  - No race conditions between check and update         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Alternative: Redis INCR is atomic                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  count = INCR(key)                                      │ │
│  │  if count == 1: EXPIRE(key, window)                    │ │
│  │  if count > limit: REJECT                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Handling Redis Failures

```
┌─────────────────────────────────────────────────────────────┐
│               Handling Redis Failures                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Option 1: Fail Open (Allow all)                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  if redis_unavailable:                                  │ │
│  │      log.warn("Rate limiter unavailable")              │ │
│  │      return ALLOW  # Better UX, risk of overload       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Option 2: Fail Closed (Reject all)                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  if redis_unavailable:                                  │ │
│  │      log.error("Rate limiter unavailable")             │ │
│  │      return REJECT  # Safer, worse UX                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Option 3: Local Fallback (Recommended)                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  if redis_unavailable:                                  │ │
│  │      return local_rate_limiter.check()                 │ │
│  │      # Use in-memory rate limiter as fallback          │ │
│  │      # Less accurate but works                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Redis Cluster for High Availability:                      │
│  • Redis Sentinel for failover                             │
│  • Redis Cluster for sharding                              │
│  • Read from replicas (eventual consistency OK)            │
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
│  │                    API Gateway                          │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │            Rate Limit Middleware                  │ │ │
│  │  │  1. Extract client identifier                    │ │ │
│  │  │  2. Determine applicable rules                   │ │ │
│  │  │  3. Check rate limit                             │ │ │
│  │  │  4. Add headers                                  │ │ │
│  │  │  5. Allow or reject                              │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                          │                                   │
│           ┌──────────────┴──────────────┐                   │
│           ▼                             ▼                   │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │  Rules Config   │           │   Redis Cluster │         │
│  │  (Rules Engine) │           │   (Rate Data)   │         │
│  └─────────────────┘           └─────────────────┘         │
│                                                              │
│  Rules Configuration:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  rules:                                                 │ │
│  │    - name: "user-api-limit"                            │ │
│  │      match:                                             │ │
│  │        path: "/api/*"                                  │ │
│  │        method: "*"                                     │ │
│  │      key: "user:${user_id}"                           │ │
│  │      limit: 1000                                       │ │
│  │      window: 3600  # 1 hour                           │ │
│  │                                                         │ │
│  │    - name: "login-limit"                               │ │
│  │      match:                                             │ │
│  │        path: "/auth/login"                             │ │
│  │        method: "POST"                                  │ │
│  │      key: "ip:${client_ip}"                           │ │
│  │      limit: 5                                          │ │
│  │      window: 60  # 1 minute                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Response Headers

```
┌─────────────────────────────────────────────────────────────┐
│                   Response Headers                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Standard Headers (RFC 6585):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  X-RateLimit-Limit: 100                                │ │
│  │  X-RateLimit-Remaining: 45                             │ │
│  │  X-RateLimit-Reset: 1640000000                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  When rate limited (HTTP 429):                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  HTTP/1.1 429 Too Many Requests                        │ │
│  │  Content-Type: application/json                        │ │
│  │  Retry-After: 30                                       │ │
│  │  X-RateLimit-Limit: 100                                │ │
│  │  X-RateLimit-Remaining: 0                              │ │
│  │  X-RateLimit-Reset: 1640000030                        │ │
│  │                                                         │ │
│  │  {                                                      │ │
│  │    "error": "rate_limit_exceeded",                     │ │
│  │    "message": "Too many requests. Please retry after   │ │
│  │                30 seconds."                            │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Rate Limiting Strategies

```
┌─────────────────────────────────────────────────────────────┐
│               Rate Limiting Strategies                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. By User ID (Authenticated)                             │
│     Key: user:{user_id}                                    │
│     Use: API rate limiting                                 │
│                                                              │
│  2. By IP Address (Anonymous)                              │
│     Key: ip:{client_ip}                                    │
│     Use: Login, public endpoints                           │
│     Caveat: NAT can group many users                       │
│                                                              │
│  3. By API Key                                              │
│     Key: apikey:{api_key}                                  │
│     Use: Third-party API access                            │
│                                                              │
│  4. By Endpoint                                             │
│     Key: endpoint:{path}:{user_id}                        │
│     Use: Different limits per endpoint                     │
│                                                              │
│  5. Tiered Limits                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Tier     │ Limit/hour │ Features                      │ │
│  │  ─────────┼────────────┼─────────────────────────────  │ │
│  │  Free     │    100     │ Basic access                  │ │
│  │  Basic    │   1,000    │ + Priority support           │ │
│  │  Pro      │  10,000    │ + Advanced features          │ │
│  │  Enterprise│ Custom    │ + SLA, dedicated support     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  6. Adaptive Rate Limiting                                  │
│     Reduce limits when system is under load               │
│     Increase limits during low traffic                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Trade-offs

```
┌─────────────────────────────────────────────────────────────┐
│                      Trade-offs                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Accuracy vs Performance                                 │
│     • Exact counting: More accurate, more overhead         │
│     • Approximate: Less accurate, faster                   │
│                                                              │
│  2. Centralized vs Distributed                              │
│     • Centralized (single Redis): Simpler, SPOF risk      │
│     • Distributed: Complex, more resilient                 │
│                                                              │
│  3. Hard vs Soft Limits                                     │
│     • Hard: Strict enforcement, better protection         │
│     • Soft: Grace period, better UX                        │
│                                                              │
│  4. Algorithm Choice                                        │
│  ┌────────────┬──────────┬──────────┬───────────────────┐  │
│  │ Algorithm  │ Memory   │ Accuracy │ Burst Handling    │  │
│  ├────────────┼──────────┼──────────┼───────────────────┤  │
│  │ Token Bucket│ O(1)    │ Good     │ Allows bursts     │  │
│  │ Leaky Bucket│ O(n)    │ Good     │ Smooths traffic   │  │
│  │ Fixed Window│ O(1)    │ Poor     │ Edge bursts       │  │
│  │ Sliding Win │ O(1)    │ Good     │ Minimal bursts    │  │
│  └────────────┴──────────┴──────────┴───────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Algorithm** | Token Bucket / Sliding Window | Balance of accuracy and performance |
| **Storage** | Redis Cluster | Fast, distributed, atomic operations |
| **Location** | API Gateway | Centralized, reusable |
| **Failure Mode** | Local fallback | Resilience with degraded accuracy |

---

[Back to System Designs](../README.md)
