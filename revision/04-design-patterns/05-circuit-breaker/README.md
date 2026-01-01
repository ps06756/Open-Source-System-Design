# Circuit Breaker

The Circuit Breaker pattern prevents cascading failures by failing fast when a downstream service is unavailable.

## Table of Contents
- [The Problem](#the-problem)
- [How It Works](#how-it-works)
- [States](#states)
- [Implementation](#implementation)
- [Related Patterns](#related-patterns)
- [Key Takeaways](#key-takeaways)

## The Problem

When a downstream service fails, naive retries can make things worse.

```
┌────────────────────────────────────────────────────────┐
│              Cascading Failure                         │
│                                                        │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐          │
│  │Service A│────▶│Service B│────▶│Service C│          │
│  │         │     │         │     │ (down)  │          │
│  └─────────┘     └─────────┘     └─────────┘          │
│       ▲               │               ✗               │
│       │               │                               │
│  Requests          Threads                            │
│  pile up           blocked                            │
│                    waiting                            │
│                                                        │
│  Without circuit breaker:                             │
│  1. C goes down                                       │
│  2. B's threads block waiting for C                  │
│  3. B's thread pool exhausted                        │
│  4. A's calls to B start failing                     │
│  5. A fails → entire system down                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## How It Works

The circuit breaker acts like an electrical circuit breaker - it "trips" when failures exceed a threshold.

```
┌────────────────────────────────────────────────────────┐
│              Circuit Breaker                           │
│                                                        │
│  ┌─────────┐     ┌─────────────┐     ┌─────────┐     │
│  │ Client  │────▶│   Circuit   │────▶│ Service │     │
│  │         │◀────│   Breaker   │◀────│         │     │
│  └─────────┘     └─────────────┘     └─────────┘     │
│                        │                              │
│                        │                              │
│                   Monitors:                           │
│                   - Failure count                     │
│                   - Failure rate                      │
│                   - Response time                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## States

### State Diagram

```
                ┌─────────────────────────────────────┐
                │                                     │
                │              CLOSED                 │
                │        (normal operation)           │
                │                                     │
                └──────────────────┬──────────────────┘
                                   │
                         Failure threshold
                              exceeded
                                   │
                                   ▼
                ┌─────────────────────────────────────┐
                │                                     │
                │               OPEN                  │
                │          (fail fast)                │
                │                                     │
                └──────────────────┬──────────────────┘
                                   │
                            Timeout expires
                                   │
                                   ▼
                ┌─────────────────────────────────────┐
                │                                     │
                │            HALF-OPEN                │
                │         (testing recovery)          │
                │                                     │
                └──────────┬──────────────────┬───────┘
                           │                  │
                     Success              Failure
                           │                  │
                           ▼                  ▼
                        CLOSED              OPEN
```

### State Details

```
┌────────────────────────────────────────────────────────┐
│                    CLOSED                              │
│                                                        │
│  - Requests flow through normally                     │
│  - Failures are counted                               │
│  - When threshold exceeded → OPEN                     │
│                                                        │
│  Threshold examples:                                   │
│  - 50% failure rate in last 10 requests              │
│  - 5 consecutive failures                             │
│  - 10 failures in 1 minute                           │
│                                                        │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│                     OPEN                               │
│                                                        │
│  - Requests fail immediately                          │
│  - No calls to downstream service                    │
│  - Returns fallback or error                         │
│  - Timer starts                                       │
│  - When timeout expires → HALF-OPEN                  │
│                                                        │
│  Timeout: typically 30-60 seconds                     │
│                                                        │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│                   HALF-OPEN                            │
│                                                        │
│  - Allow limited requests through                     │
│  - Testing if service recovered                       │
│  - If success → CLOSED                               │
│  - If failure → OPEN (reset timer)                   │
│                                                        │
│  Typically allow 1-3 test requests                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Implementation

### Basic Circuit Breaker

```python
from enum import Enum
from datetime import datetime, timedelta
import threading

class State(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold=5,
        success_threshold=2,
        timeout=30
    ):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout

        self.state = State.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self.lock = threading.Lock()

    def call(self, func, *args, **kwargs):
        with self.lock:
            if self.state == State.OPEN:
                if self._should_try_reset():
                    self.state = State.HALF_OPEN
                else:
                    raise CircuitOpenException()

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        with self.lock:
            if self.state == State.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.success_threshold:
                    self.state = State.CLOSED
                    self._reset_counts()
            else:
                self._reset_counts()

    def _on_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = datetime.now()

            if self.state == State.HALF_OPEN:
                self.state = State.OPEN
                self._reset_counts()
            elif self.failure_count >= self.failure_threshold:
                self.state = State.OPEN

    def _should_try_reset(self):
        return (
            datetime.now() - self.last_failure_time
        ) > timedelta(seconds=self.timeout)

    def _reset_counts(self):
        self.failure_count = 0
        self.success_count = 0
```

### Usage

```python
# Create circuit breaker
payment_cb = CircuitBreaker(
    failure_threshold=5,
    success_threshold=2,
    timeout=30
)

# Use with external call
def process_payment(order):
    try:
        result = payment_cb.call(
            payment_service.charge,
            order.customer_id,
            order.total
        )
        return result
    except CircuitOpenException:
        # Fallback behavior
        return queue_for_later(order)
    except PaymentException as e:
        # Handle actual payment failure
        raise
```

### Configuration

```python
# Conservative (strict)
CircuitBreaker(
    failure_threshold=3,    # Trip after 3 failures
    success_threshold=5,    # Need 5 successes to close
    timeout=60              # Wait 60s before trying again
)

# Lenient
CircuitBreaker(
    failure_threshold=10,
    success_threshold=1,
    timeout=10
)

# Sliding window (more sophisticated)
CircuitBreaker(
    failure_rate_threshold=50,  # 50% failure rate
    minimum_requests=10,        # Need 10 requests to evaluate
    window_size=60,             # 60 second window
    timeout=30
)
```

## Related Patterns

### Bulkhead

Isolate components to prevent failure spread.

```
┌────────────────────────────────────────────────────────┐
│                    Bulkhead                            │
│                                                        │
│  Without bulkhead:                                     │
│  ┌─────────────────────────────────────────┐          │
│  │        Single Thread Pool (100)          │          │
│  │  All services share pool                │          │
│  │  One slow service blocks all            │          │
│  └─────────────────────────────────────────┘          │
│                                                        │
│  With bulkhead:                                        │
│  ┌───────────────┐ ┌───────────────┐ ┌─────────────┐ │
│  │ Service A (30)│ │ Service B (30)│ │ Service C   │ │
│  │               │ │               │ │    (40)     │ │
│  └───────────────┘ └───────────────┘ └─────────────┘ │
│  Slow Service A can only exhaust its own pool        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Retry with Backoff

Retry failed requests with increasing delays.

```python
def retry_with_backoff(func, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientException:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # Exponential
            delay += random.uniform(0, 1)         # Jitter
            time.sleep(delay)
```

### Timeout

Always set timeouts on external calls.

```python
# HTTP client timeout
response = requests.get(url, timeout=5)  # 5 seconds

# Database query timeout
cursor.execute("SET statement_timeout = '5s'")

# Combined pattern
@circuit_breaker
@timeout(seconds=5)
@retry(max_attempts=3)
def call_external_service():
    pass
```

### Fallback

Provide alternative when circuit is open.

```python
def get_product_recommendations(user_id):
    try:
        return circuit_breaker.call(
            recommendation_service.get,
            user_id
        )
    except CircuitOpenException:
        # Fallback to cached or default recommendations
        return get_cached_recommendations(user_id)
    except Exception:
        # Fallback on any error
        return get_popular_products()
```

## Libraries

| Language | Library |
|----------|---------|
| Java | Resilience4j, Hystrix (deprecated) |
| Python | pybreaker, circuitbreaker |
| Go | sony/gobreaker |
| JavaScript | opossum |
| .NET | Polly |

### Resilience4j Example (Java)

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50)
    .waitDurationInOpenState(Duration.ofSeconds(30))
    .slidingWindowSize(10)
    .build();

CircuitBreaker circuitBreaker = CircuitBreaker.of("payment", config);

Supplier<Payment> decorated = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> paymentService.charge(order));

Try<Payment> result = Try.ofSupplier(decorated)
    .recover(throwable -> fallbackPayment(order));
```

## Key Takeaways

1. **Fail fast** when downstream is unhealthy
2. **Three states**: Closed (normal), Open (failing), Half-Open (testing)
3. **Configure thresholds** based on your tolerance
4. **Always provide fallback** behavior
5. **Combine with** timeout, retry, and bulkhead
6. **Monitor circuit state** for visibility

## Practice Questions

1. How would you configure a circuit breaker for a payment service?
2. What's the difference between circuit breaker and retry?
3. How do you test circuit breaker behavior?
4. Design a system with multiple circuit breakers.

## Further Reading

- [Circuit Breaker by Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/)
- [Resilience4j Documentation](https://resilience4j.readme.io/)

---

Next: [Sidecar & Ambassador](../06-sidecar-ambassador/README.md)
