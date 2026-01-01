# Rate Limiter

Rate limiting protects services from abuse and ensures fair resource usage by limiting the number of requests a client can make in a given time period.

## Table of Contents
1. [Why Rate Limiting?](#why-rate-limiting)
2. [Rate Limiting Algorithms](#rate-limiting-algorithms)
3. [Distributed Rate Limiting](#distributed-rate-limiting)
4. [Implementation Considerations](#implementation-considerations)
5. [Interview Questions](#interview-questions)

---

## Why Rate Limiting?

```
┌─────────────────────────────────────────────────────────────┐
│                   Why Rate Limiting?                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without Rate Limiting:                                      │
│  ┌────────┐  1000 req/s   ┌─────────┐                       │
│  │ Abuser │──────────────▶│ Service │ ← Overwhelmed!        │
│  └────────┘               └─────────┘                       │
│  ┌────────┐     1 req/s   Can't serve legitimate users      │
│  │ Normal │──────────────▶                                  │
│  └────────┘                                                  │
│                                                              │
│  With Rate Limiting:                                         │
│  ┌────────┐  1000 req/s   ┌───────────┐    ┌─────────┐     │
│  │ Abuser │──────────────▶│   Rate    │───▶│ Service │     │
│  └────────┘   (blocked)   │  Limiter  │    └─────────┘     │
│  ┌────────┐     1 req/s   │           │         ↑          │
│  │ Normal │──────────────▶│  100/min  │─────────┘          │
│  └────────┘   (allowed)   └───────────┘                     │
│                                                              │
│  Protects against:                                           │
│  • DoS attacks                                               │
│  • Brute force attacks                                       │
│  • Resource hogging                                          │
│  • API abuse                                                 │
│  • Runaway scripts/bugs                                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Rate Limiting Algorithms

### 1. Token Bucket

```
┌─────────────────────────────────────────────────────────────┐
│                      Token Bucket                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  How it works:                                               │
│  ┌──────────────────────────────────────┐                   │
│  │           Token Bucket               │                   │
│  │  ┌───┬───┬───┬───┬───┬───┬───┬───┐  │                   │
│  │  │ ○ │ ○ │ ○ │ ○ │ ○ │   │   │   │  │  ← Max capacity   │
│  │  └───┴───┴───┴───┴───┴───┴───┴───┘  │                   │
│  │       ↑                              │                   │
│  │       │ Tokens added at fixed rate   │                   │
│  │       │ (e.g., 10 tokens/second)     │                   │
│  └──────────────────────────────────────┘                   │
│            │                                                 │
│            ▼ Each request takes 1 token                     │
│  ┌──────────────────────────────────────┐                   │
│  │  Request → Has token? → Yes → Allow  │                   │
│  │                      → No  → Reject  │                   │
│  └──────────────────────────────────────┘                   │
│                                                              │
│  Properties:                                                 │
│  • Allows bursts up to bucket capacity                      │
│  • Smooth rate over time                                    │
│  • Memory: O(1) per user                                    │
│                                                              │
│  Parameters:                                                 │
│  • Bucket size (max tokens)                                 │
│  • Refill rate (tokens per second)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:**
```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow_request(self):
        self._refill()

        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
```

### 2. Leaky Bucket

```
┌─────────────────────────────────────────────────────────────┐
│                      Leaky Bucket                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  How it works:                                               │
│                                                              │
│  Requests come in (variable rate)                           │
│       │ │ │ │ │ │                                           │
│       ▼ ▼ ▼ ▼ ▼ ▼                                           │
│  ┌───────────────────┐                                      │
│  │      Bucket       │ ← Queue with fixed capacity          │
│  │  ┌───┬───┬───┐   │                                      │
│  │  │ R │ R │ R │   │   If full → reject new requests      │
│  │  └───┴───┴───┘   │                                      │
│  │        │         │                                       │
│  └────────│─────────┘                                       │
│           │                                                  │
│           ▼  Leaks at constant rate                         │
│      ┌─────────┐                                            │
│      │ Service │  Requests processed at fixed rate          │
│      └─────────┘                                            │
│                                                              │
│  Difference from Token Bucket:                              │
│  • Token Bucket: controls average rate, allows bursts       │
│  • Leaky Bucket: smooths output to constant rate           │
│                                                              │
│  Properties:                                                 │
│  • Constant output rate (no bursts)                        │
│  • Queue-based implementation                               │
│  • Good for traffic shaping                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3. Fixed Window Counter

```
┌─────────────────────────────────────────────────────────────┐
│                  Fixed Window Counter                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Time divided into fixed windows:                           │
│                                                              │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │  Window 1   │  Window 2   │  Window 3   │               │
│  │  00:00-01:00│  01:00-02:00│  02:00-03:00│               │
│  │  Count: 95  │  Count: 42  │  Count: 0   │               │
│  │  Limit: 100 │  Limit: 100 │  Limit: 100 │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                              │
│  Simple logic:                                               │
│  if count[current_window] < limit:                         │
│      count[current_window]++                               │
│      allow()                                                │
│  else:                                                      │
│      reject()                                               │
│                                                              │
│  Problem - Burst at window edges:                          │
│  ┌─────────────┬─────────────┐                             │
│  │  Window 1   │  Window 2   │                             │
│  │     ....100 │ 100....     │  ← 200 requests in 1 second!│
│  │  00:59-01:00│ 01:00-01:01 │                             │
│  └─────────────┴─────────────┘                             │
│                                                              │
│  Pros: Simple, memory efficient                            │
│  Cons: Burst at window boundaries                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4. Sliding Window Log

```
┌─────────────────────────────────────────────────────────────┐
│                   Sliding Window Log                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Store timestamp of each request:                           │
│                                                              │
│  Request log for user_123:                                  │
│  [12:00:15, 12:00:23, 12:00:45, 12:00:58, 12:01:02]        │
│                                                              │
│  On new request at 12:01:10:                               │
│  1. Remove entries older than window (1 min ago = 12:00:10)│
│     [12:00:15, 12:00:23, 12:00:45, 12:00:58, 12:01:02]     │
│                                                              │
│  2. Count remaining: 5                                      │
│  3. If count < limit (100): allow and add timestamp        │
│                                                              │
│  ◄────────── 1 minute window ──────────►                   │
│  │                                      │                   │
│  12:00:10                           12:01:10               │
│                                                              │
│  Pros:                                                      │
│  • Very accurate                                            │
│  • No boundary issues                                       │
│                                                              │
│  Cons:                                                      │
│  • Memory intensive (stores all timestamps)                │
│  • O(n) cleanup per request                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5. Sliding Window Counter (Recommended)

```
┌─────────────────────────────────────────────────────────────┐
│                Sliding Window Counter                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Combines Fixed Window + Sliding Window Log benefits        │
│                                                              │
│  ┌─────────────────┬─────────────────┐                     │
│  │  Previous Window │  Current Window │                     │
│  │    Count: 80     │    Count: 20    │                     │
│  │   (70% passed)   │   (30% in)      │                     │
│  └─────────────────┴─────────────────┘                     │
│        │                    │                               │
│        │◄── 70% ──►│◄─ 30% ─│                              │
│                    │                                         │
│                 Current time                                │
│                                                              │
│  Weighted count = prev_count × prev_weight + curr_count    │
│                 = 80 × 0.7 + 20                            │
│                 = 56 + 20 = 76                             │
│                                                              │
│  If weighted_count < limit (100): allow                    │
│                                                              │
│  Pros:                                                      │
│  • Smooth rate limiting                                    │
│  • Memory efficient (just 2 counters per window)           │
│  • Good approximation of sliding window                    │
│                                                              │
│  This is what most production systems use!                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:**
```python
class SlidingWindowCounter:
    def __init__(self, limit, window_size_seconds):
        self.limit = limit
        self.window_size = window_size_seconds
        self.prev_count = 0
        self.curr_count = 0
        self.curr_window_start = self._get_window_start()

    def allow_request(self):
        now = time.time()
        window_start = self._get_window_start()

        # New window started
        if window_start != self.curr_window_start:
            self.prev_count = self.curr_count
            self.curr_count = 0
            self.curr_window_start = window_start

        # Calculate weighted count
        elapsed = now - window_start
        prev_weight = 1 - (elapsed / self.window_size)
        weighted_count = self.prev_count * prev_weight + self.curr_count

        if weighted_count < self.limit:
            self.curr_count += 1
            return True
        return False

    def _get_window_start(self):
        return int(time.time() / self.window_size) * self.window_size
```

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity |
|-----------|--------|----------|----------------|------------|
| Token Bucket | O(1) | Good | Allows controlled bursts | Simple |
| Leaky Bucket | O(n) | Excellent | No bursts (smooths) | Medium |
| Fixed Window | O(1) | Poor at edges | Edge bursts | Simple |
| Sliding Log | O(n) | Excellent | No bursts | Medium |
| Sliding Counter | O(1) | Good | Minimal edge issues | Simple |

---

## Distributed Rate Limiting

```
┌─────────────────────────────────────────────────────────────┐
│              Distributed Rate Limiting                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Challenge: Multiple servers need consistent rate limiting  │
│                                                              │
│  ┌────────┐     ┌──────────┐                               │
│  │Client  │────▶│ Server 1 │──┐                            │
│  └────────┘     └──────────┘  │   Each server has its own  │
│                               │   counter = inconsistent!   │
│  ┌────────┐     ┌──────────┐  │                            │
│  │Client  │────▶│ Server 2 │──┤                            │
│  └────────┘     └──────────┘  │                            │
│                               │                             │
│  ┌────────┐     ┌──────────┐  │                            │
│  │Client  │────▶│ Server 3 │──┘                            │
│  └────────┘     └──────────┘                               │
│                                                              │
│  Solution: Centralized rate limit store                     │
│                                                              │
│  ┌────────┐     ┌──────────┐     ┌─────────────────┐       │
│  │Client  │────▶│ Server 1 │────▶│                 │       │
│  └────────┘     └──────────┘     │   Redis Cluster │       │
│                                   │                 │       │
│  ┌────────┐     ┌──────────┐     │  user:123:count │       │
│  │Client  │────▶│ Server 2 │────▶│  user:456:count │       │
│  └────────┘     └──────────┘     │  ip:1.2.3.4:cnt │       │
│                                   │                 │       │
│  ┌────────┐     ┌──────────┐     └─────────────────┘       │
│  │Client  │────▶│ Server 3 │────────────┘                  │
│  └────────┘     └──────────┘                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Redis-Based Rate Limiting

```
┌─────────────────────────────────────────────────────────────┐
│              Redis Rate Limiting                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Sliding Window Counter with Redis:                         │
│                                                              │
│  # Lua script for atomic rate limiting                      │
│  local key = KEYS[1]                                        │
│  local limit = tonumber(ARGV[1])                           │
│  local window = tonumber(ARGV[2])                          │
│                                                              │
│  local current = redis.call('INCR', key)                   │
│  if current == 1 then                                       │
│      redis.call('EXPIRE', key, window)                     │
│  end                                                         │
│                                                              │
│  if current > limit then                                    │
│      return 0  -- rejected                                  │
│  end                                                         │
│  return 1  -- allowed                                       │
│                                                              │
│  Key format: rate_limit:{user_id}:{window_timestamp}       │
│                                                              │
│  Why Lua script?                                            │
│  • Atomic execution (no race conditions)                   │
│  • Single round trip to Redis                              │
│  • Guaranteed consistency                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Race Condition Problem

```
┌─────────────────────────────────────────────────────────────┐
│                 Race Condition Problem                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without atomic operations:                                 │
│                                                              │
│  Server 1              Redis              Server 2          │
│     │                    │                    │             │
│     │─── GET count ─────▶│                    │             │
│     │◀── count = 99 ─────│                    │             │
│     │                    │◀── GET count ──────│             │
│     │                    │─── count = 99 ────▶│             │
│     │                    │                    │             │
│     │─── SET count=100 ─▶│                    │             │
│     │                    │◀── SET count=100 ──│             │
│     │                    │                    │             │
│  Both allowed! But limit is 100!                            │
│                                                              │
│  Solution: Use INCR or Lua scripts                          │
│                                                              │
│  Server 1              Redis              Server 2          │
│     │                    │                    │             │
│     │─── INCR count ────▶│                    │             │
│     │◀── returns 100 ────│                    │             │
│     │   (allowed)        │◀── INCR count ─────│             │
│     │                    │─── returns 101 ───▶│             │
│     │                    │   (rejected)       │             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Considerations

```
┌─────────────────────────────────────────────────────────────┐
│              Implementation Considerations                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Rate Limit by What?                                     │
│     ┌────────────────────────────────────────────────────┐ │
│     │ • User ID    - Most common, requires auth          │ │
│     │ • API Key    - For API access control              │ │
│     │ • IP Address - Anonymous users, DDoS protection    │ │
│     │ • Endpoint   - Different limits per API            │ │
│     │ • Combined   - IP + User for defense in depth     │ │
│     └────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Where to Rate Limit?                                    │
│     ┌────────────────────────────────────────────────────┐ │
│     │ Client → CDN → Load Balancer → API Gateway → App  │ │
│     │          ↑           ↑              ↑         ↑   │ │
│     │       DDoS      L4 limits      L7 limits  Custom │ │
│     │    protection                              rules  │ │
│     └────────────────────────────────────────────────────┘ │
│     Best: Multiple layers with different limits           │
│                                                              │
│  3. Response Headers (Standard Practice):                   │
│     ┌────────────────────────────────────────────────────┐ │
│     │ X-RateLimit-Limit: 100                             │ │
│     │ X-RateLimit-Remaining: 45                          │ │
│     │ X-RateLimit-Reset: 1640000000 (Unix timestamp)     │ │
│     │                                                     │ │
│     │ When rate limited:                                 │ │
│     │ HTTP 429 Too Many Requests                         │ │
│     │ Retry-After: 30 (seconds)                          │ │
│     └────────────────────────────────────────────────────┘ │
│                                                              │
│  4. Graceful Degradation:                                   │
│     ┌────────────────────────────────────────────────────┐ │
│     │ If rate limiter fails (Redis down):               │ │
│     │ Option A: Allow all requests (availability)        │ │
│     │ Option B: Reject all requests (safety)            │ │
│     │ Option C: Local fallback limiter                  │ │
│     │                                                     │ │
│     │ Usually: Allow with local fallback + alert        │ │
│     └────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Rate Limiting Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                Rate Limiting Strategies                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Hard Rate Limiting                                      │
│     Strict limit, reject once exceeded                     │
│     Use: Security-critical endpoints (login, payments)     │
│                                                              │
│  2. Soft Rate Limiting                                      │
│     Allow burst, then throttle                             │
│     Use: User-facing APIs where UX matters                 │
│                                                              │
│  3. Tiered Rate Limiting                                    │
│     ┌────────────────────────────────────────────────────┐ │
│     │ Tier        │ Limit/min │ Price                    │ │
│     ├─────────────┼───────────┼──────────────────────────│ │
│     │ Free        │ 100       │ $0                       │ │
│     │ Basic       │ 1,000     │ $10/mo                   │ │
│     │ Pro         │ 10,000    │ $100/mo                  │ │
│     │ Enterprise  │ Unlimited │ Custom                   │ │
│     └────────────────────────────────────────────────────┘ │
│                                                              │
│  4. Adaptive Rate Limiting                                  │
│     Adjust limits based on server load                     │
│     Reduce limits during high traffic                      │
│                                                              │
│  5. Geographic Rate Limiting                                │
│     Different limits per region                            │
│     Higher limits for paying regions                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is rate limiting and why is it important?
2. Explain the token bucket algorithm.

### Intermediate
3. How would you implement distributed rate limiting?
4. Compare sliding window counter vs fixed window counter.

### Advanced
5. Design a rate limiter for a high-traffic API handling 1M requests/second.

---

[← Previous: API Gateway](../09-api-gateway/README.md) | [Back to Building Blocks](../README.md)
