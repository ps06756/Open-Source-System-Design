# Circuit Breaker Pattern

The Circuit Breaker pattern prevents cascading failures in distributed systems by failing fast when a service is unhealthy, giving it time to recover.

## Table of Contents
1. [Why Circuit Breakers?](#why-circuit-breakers)
2. [How It Works](#how-it-works)
3. [Implementation](#implementation)
4. [Related Patterns](#related-patterns)
5. [Interview Questions](#interview-questions)

---

## Why Circuit Breakers?

```
┌─────────────────────────────────────────────────────────────┐
│                 Why Circuit Breakers?                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Cascading Failures                                │
│                                                              │
│  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐      │
│  │Service │───▶│Service │───▶│Service │───▶│Service │      │
│  │   A    │    │   B    │    │   C    │    │   D    │      │
│  └────────┘    └────────┘    └────────┘    └────────┘      │
│                                               ↓             │
│                                            FAILING          │
│                                                              │
│  Without Circuit Breaker:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. D is slow/failing                                  │ │
│  │  2. C waits for D, C's threads exhausted               │ │
│  │  3. B waits for C, B's threads exhausted               │ │
│  │  4. A waits for B, A's threads exhausted               │ │
│  │  5. ENTIRE SYSTEM DOWN!                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  With Circuit Breaker:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. D is slow/failing                                  │ │
│  │  2. C detects failures, opens circuit breaker          │ │
│  │  3. C fails fast, returns fallback                     │ │
│  │  4. B, A continue working with fallback               │ │
│  │  5. D has time to recover                              │ │
│  │  6. Circuit closes, normal operation resumes           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Named after electrical circuit breakers that prevent      │
│  electrical fires by breaking the circuit!                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│              Circuit Breaker States                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Three States:                                               │
│                                                              │
│  ┌──────────────┐                                           │
│  │    CLOSED    │  ← Normal operation                       │
│  │              │    Requests pass through                  │
│  │   ○ ○ ○ ○    │    Counting failures                     │
│  └──────┬───────┘                                           │
│         │ Failure threshold exceeded                        │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │     OPEN     │  ← Circuit broken                         │
│  │              │    Requests fail immediately              │
│  │   ─ ─ ─ ─    │    No calls to failing service           │
│  └──────┬───────┘                                           │
│         │ Timeout expires                                   │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │  HALF-OPEN   │  ← Testing recovery                       │
│  │              │    Limited requests allowed               │
│  │   ○ ─ ○ ─    │    Success → CLOSED, Failure → OPEN     │
│  └──────────────┘                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### State Transitions

```
┌─────────────────────────────────────────────────────────────┐
│                  State Transitions                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    success                                   │
│           ┌────────────────────────────┐                    │
│           │                            │                    │
│           ▼                            │                    │
│      ┌────────┐  failure threshold  ┌──┴─────┐              │
│      │ CLOSED │────────────────────▶│  OPEN  │              │
│      └────────┘                     └────────┘              │
│           ▲                              │                   │
│           │                              │ timeout           │
│           │                              ▼                   │
│           │    success             ┌───────────┐            │
│           └────────────────────────│ HALF-OPEN │            │
│                                    └───────────┘            │
│                                          │                   │
│                                          │ failure           │
│                                          ▼                   │
│                                    ┌──────────┐             │
│                                    │   OPEN   │             │
│                                    └──────────┘             │
│                                                              │
│  Parameters:                                                 │
│  • Failure threshold: 5 failures in 10 seconds             │
│  • Open timeout: 30 seconds                                 │
│  • Half-open requests: 3                                    │
│  • Success threshold: 2 successes to close                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation

```
┌─────────────────────────────────────────────────────────────┐
│              Circuit Breaker Implementation                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Basic Implementation:                                      │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  class CircuitBreaker:                                  │ │
│  │    def __init__(self,                                  │ │
│  │                 failure_threshold=5,                   │ │
│  │                 reset_timeout=30):                     │ │
│  │      self.state = "CLOSED"                            │ │
│  │      self.failures = 0                                │ │
│  │      self.last_failure_time = None                    │ │
│  │      self.failure_threshold = failure_threshold       │ │
│  │      self.reset_timeout = reset_timeout               │ │
│  │                                                         │ │
│  │    def call(self, func):                              │ │
│  │      if self.state == "OPEN":                         │ │
│  │        if self._timeout_expired():                    │ │
│  │          self.state = "HALF_OPEN"                     │ │
│  │        else:                                           │ │
│  │          raise CircuitOpenError()                     │ │
│  │                                                         │ │
│  │      try:                                              │ │
│  │        result = func()                                │ │
│  │        self._on_success()                             │ │
│  │        return result                                  │ │
│  │      except Exception as e:                           │ │
│  │        self._on_failure()                             │ │
│  │        raise                                           │ │
│  │                                                         │ │
│  │    def _on_success(self):                             │ │
│  │      self.failures = 0                                │ │
│  │      self.state = "CLOSED"                            │ │
│  │                                                         │ │
│  │    def _on_failure(self):                             │ │
│  │      self.failures += 1                               │ │
│  │      if self.failures >= self.failure_threshold:      │ │
│  │        self.state = "OPEN"                            │ │
│  │        self.last_failure_time = time.now()            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Fallback Strategies

```
┌─────────────────────────────────────────────────────────────┐
│                  Fallback Strategies                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  When circuit is open, use fallback:                        │
│                                                              │
│  1. Default Value                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def get_recommendations():                             │ │
│  │    try:                                                 │ │
│  │      return recommendation_service.get()               │ │
│  │    except CircuitOpenError:                            │ │
│  │      return DEFAULT_RECOMMENDATIONS  # Popular items   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Cached Value                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def get_product(id):                                  │ │
│  │    try:                                                 │ │
│  │      return product_service.get(id)                    │ │
│  │    except CircuitOpenError:                            │ │
│  │      return cache.get(f"product:{id}")  # Stale OK     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Graceful Degradation                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def get_page():                                        │ │
│  │    product = product_service.get(id)                   │ │
│  │    try:                                                 │ │
│  │      reviews = review_service.get(id)                  │ │
│  │    except CircuitOpenError:                            │ │
│  │      reviews = None  # Show page without reviews       │ │
│  │    return render(product, reviews)                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  4. Queue for Later                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  def send_notification(message):                       │ │
│  │    try:                                                 │ │
│  │      notification_service.send(message)                │ │
│  │    except CircuitOpenError:                            │ │
│  │      queue.add(message)  # Retry later                 │ │
│  │      return "Notification queued"                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Popular Libraries

```
┌─────────────────────────────────────────────────────────────┐
│                   Popular Libraries                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Java:                                                       │
│  • Resilience4j (recommended)                              │
│  • Hystrix (Netflix, deprecated)                           │
│                                                              │
│  Python:                                                     │
│  • pybreaker                                                │
│  • tenacity                                                 │
│                                                              │
│  JavaScript:                                                 │
│  • opossum                                                  │
│  • cockatiel                                                │
│                                                              │
│  Go:                                                         │
│  • sony/gobreaker                                          │
│  • afex/hystrix-go                                         │
│                                                              │
│  Service Mesh (handles it for you):                        │
│  • Istio                                                    │
│  • Linkerd                                                  │
│  • Envoy (sidecar proxy)                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Related Patterns

```
┌─────────────────────────────────────────────────────────────┐
│                   Related Patterns                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Retry Pattern                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Automatically retry failed operations                 │ │
│  │                                                         │ │
│  │  Request ──▶ Fail ──▶ Wait ──▶ Retry ──▶ Success      │ │
│  │                                                         │ │
│  │  Use with exponential backoff:                         │ │
│  │  Retry 1: wait 1s                                      │ │
│  │  Retry 2: wait 2s                                      │ │
│  │  Retry 3: wait 4s (+ jitter)                          │ │
│  │                                                         │ │
│  │  Combine: Retry inside circuit breaker                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Timeout Pattern                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Don't wait forever for response                       │ │
│  │                                                         │ │
│  │  Request ──▶ 5 second timeout ──▶ TimeoutError        │ │
│  │                                                         │ │
│  │  Set appropriate timeouts:                             │ │
│  │  • Connection timeout: 1-3s                            │ │
│  │  • Read timeout: based on operation                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Bulkhead Pattern                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Isolate failures to prevent full system failure       │ │
│  │                                                         │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐          │ │
│  │  │ Service A │  │ Service B │  │ Service C │          │ │
│  │  │ Pool: 10  │  │ Pool: 10  │  │ Pool: 10  │          │ │
│  │  └───────────┘  └───────────┘  └───────────┘          │ │
│  │                                                         │ │
│  │  If A's pool exhausted, B and C unaffected            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Combined Pattern:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Request ──▶ Bulkhead ──▶ Timeout ──▶ Circuit Breaker │ │
│  │                                ──▶ Retry              │ │
│  │                                ──▶ Fallback           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Rate Limiting vs Circuit Breaker

```
┌─────────────────────────────────────────────────────────────┐
│          Rate Limiting vs Circuit Breaker                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────┬───────────────────────────────┐ │
│  │    Rate Limiting       │     Circuit Breaker           │ │
│  ├────────────────────────┼───────────────────────────────┤ │
│  │ Protects service FROM  │ Protects caller FROM         │ │
│  │ too many requests      │ failing service              │ │
│  │                        │                               │ │
│  │ Proactive (prevents    │ Reactive (responds to        │ │
│  │ overload)              │ failures)                     │ │
│  │                        │                               │ │
│  │ Based on request       │ Based on failure             │ │
│  │ count/rate             │ count/percentage              │ │
│  │                        │                               │ │
│  │ Always applied         │ Opens when problems          │ │
│  │                        │ detected                      │ │
│  └────────────────────────┴───────────────────────────────┘ │
│                                                              │
│  Use both together for resilient systems!                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the circuit breaker pattern?
2. What are the three states of a circuit breaker?

### Intermediate
3. How would you choose circuit breaker thresholds?
4. What fallback strategies would you use?

### Advanced
5. How do you implement circuit breakers in a microservices architecture?
6. Compare circuit breaker with bulkhead pattern.

---

[← Previous: Saga Pattern](../04-saga-pattern/README.md) | [Next: Sidecar & Service Mesh →](../06-sidecar-service-mesh/README.md)
