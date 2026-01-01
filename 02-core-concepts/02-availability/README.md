# Availability & Reliability

Availability and reliability are critical for production systems. Understanding these concepts helps you design systems that users can depend on.

## Table of Contents
- [Definitions](#definitions)
- [Measuring Availability](#measuring-availability)
- [SLAs, SLOs, and SLIs](#slas-slos-and-slis)
- [Achieving High Availability](#achieving-high-availability)
- [Failure Modes](#failure-modes)
- [Fault Tolerance Patterns](#fault-tolerance-patterns)
- [Key Takeaways](#key-takeaways)

## Definitions

### Availability

The proportion of time a system is operational and accessible.

```
                    Uptime
Availability = ─────────────────────
               Uptime + Downtime
```

### Reliability

The probability that a system performs its intended function without failure over a period of time.

### Difference

| Concept | Focus | Example |
|---------|-------|---------|
| Availability | Uptime | "The website was up 99.9% of the time" |
| Reliability | Correctness | "All transactions were processed correctly" |

A system can be available but unreliable (serving errors) or reliable but unavailable (down for maintenance).

## Measuring Availability

### The "Nines" of Availability

| Availability | Downtime/Year | Downtime/Month | Downtime/Day |
|--------------|---------------|----------------|--------------|
| 90% (one nine) | 36.5 days | 72 hours | 2.4 hours |
| 99% (two nines) | 3.65 days | 7.2 hours | 14.4 minutes |
| 99.9% (three nines) | 8.76 hours | 43.2 minutes | 1.44 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.32 minutes | 8.6 seconds |
| 99.999% (five nines) | 5.26 minutes | 26 seconds | 0.86 seconds |
| 99.9999% (six nines) | 31.5 seconds | 2.6 seconds | 86 milliseconds |

### What Major Services Target

| Service | Target | Notes |
|---------|--------|-------|
| AWS EC2 | 99.99% | Per region |
| Google Cloud | 99.95% | Most services |
| Azure | 99.95-99.99% | Varies by service |
| Stripe | 99.99% | API availability |
| Slack | 99.99% | Core functionality |

### Availability in Series vs Parallel

**Series (dependent components)**:
```
A ──▶ B ──▶ C

Total = A × B × C
If each is 99.9%: 0.999 × 0.999 × 0.999 = 99.7%
```

**Parallel (redundant components)**:
```
    ┌── A ──┐
────┤       ├────
    └── B ──┘

Total = 1 - (1-A) × (1-B)
If each is 99%: 1 - (0.01 × 0.01) = 99.99%
```

## SLAs, SLOs, and SLIs

### Service Level Indicator (SLI)

A quantitative measure of service behavior.

**Examples:**
- Request latency (p99 < 200ms)
- Error rate (< 0.1%)
- Throughput (> 1000 RPS)
- Availability (> 99.9%)

### Service Level Objective (SLO)

A target value for an SLI.

```
SLO: 99.9% of requests should complete in < 200ms
     over a rolling 30-day window
```

### Service Level Agreement (SLA)

A contract with consequences for missing objectives.

```
SLA: 99.9% availability per month
     If missed: 10% service credit
     If below 99%: 25% service credit
```

### Relationship

```
┌─────────────────────────────────────────────────────────┐
│                         SLA                             │
│   (Contract with customers)                             │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │                     SLO                          │   │
│   │   (Internal targets, tighter than SLA)          │   │
│   │                                                  │   │
│   │   ┌─────────────────────────────────────────┐   │   │
│   │   │                 SLI                      │   │   │
│   │   │   (Actual measurements)                 │   │   │
│   │   └─────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Error Budgets

The acceptable amount of unreliability.

```
If SLO is 99.9%:
Error Budget = 100% - 99.9% = 0.1%

Per month = 0.1% × 43,200 minutes = 43.2 minutes of downtime allowed
```

**Uses:**
- Balance reliability vs feature velocity
- If budget exhausted, freeze deployments
- Invest in reliability when budget is low

## Achieving High Availability

### Redundancy

Eliminate single points of failure:

```
Single Point of Failure:
┌────────┐     ┌────────┐     ┌────────┐
│ Client │────▶│ Server │────▶│   DB   │
└────────┘     └────────┘     └────────┘
                   ✗              ✗

With Redundancy:
              ┌──────────────┐
              │     LB       │
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │Server 1│  │Server 2│  │Server 3│
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     ▼
              ┌──────────────┐
              │  DB Primary  │────┐
              └──────────────┘    │ Replication
                                  ▼
                           ┌──────────────┐
                           │  DB Replica  │
                           └──────────────┘
```

### Geographic Distribution

```
┌─────────────────────────────────────────────────────────┐
│                    Global Load Balancer                  │
└────────────────────────┬────────────────────────────────┘
                         │
       ┌─────────────────┼─────────────────┐
       ▼                 ▼                 ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  US-East    │   │  EU-West    │   │  AP-Tokyo   │
│  Region     │   │  Region     │   │  Region     │
└─────────────┘   └─────────────┘   └─────────────┘
```

### Health Checks

```
┌──────────────┐
│ Load Balancer│
└──────┬───────┘
       │
       │  Health check every 10s
       │  /health endpoint
       │
       ▼
┌─────────────┐    ✓ Healthy ──▶ Route traffic
│   Server    │
└─────────────┘    ✗ Unhealthy ──▶ Remove from pool
```

### Graceful Degradation

When under stress, reduce functionality rather than fail completely:

```
Normal:     Full features, personalization, recommendations
Degraded:   Core features only, no personalization
Minimal:    Read-only mode, cached content
```

## Failure Modes

### Types of Failures

| Type | Description | Example |
|------|-------------|---------|
| Crash | Process terminates | OOM kill |
| Omission | Fails to respond | Network partition |
| Timing | Response too slow | Overloaded server |
| Response | Wrong response | Bug, corruption |
| Byzantine | Arbitrary behavior | Compromised node |

### MTBF and MTTR

```
      ┌───────┐                    ┌───────┐
      │ Fail  │                    │ Fail  │
──────┴───────┴────────────────────┴───────┴──────▶
      ◀──────── MTBF ────────────▶
            (Mean Time Between Failures)

      ┌───────┐
      │ Fail  │
──────┴───────┴───────────────────────────────────▶
      ◀─ MTTR ─▶
   (Mean Time To Recovery)

Availability = MTBF / (MTBF + MTTR)
```

**To improve availability:**
- Increase MTBF (better hardware, fewer bugs)
- Decrease MTTR (faster detection, automated recovery)

## Fault Tolerance Patterns

### 1. Retry with Exponential Backoff

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception:
            if attempt == max_retries - 1:
                raise
            wait = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait)
```

```
Attempt 1: Wait 0s
Attempt 2: Wait 1s + jitter
Attempt 3: Wait 2s + jitter
Attempt 4: Wait 4s + jitter
Attempt 5: Wait 8s + jitter
```

### 2. Circuit Breaker

```
         Closed                 Open                  Half-Open
    ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
    │   Normal     │       │   Fail Fast  │       │    Test      │
    │  Operation   │       │              │       │              │
    └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
           │                      │                      │
           │ Failures             │ Timeout              │
           │ exceed               │ expires              │ Success
           │ threshold            │                      │
           ▼                      ▼                      ▼
      ┌─────────┐            ┌─────────┐            ┌─────────┐
      │  Open   │───────────▶│Half-Open│───────────▶│ Closed  │
      └─────────┘            └─────────┘            └─────────┘
                                  │
                                  │ Failure
                                  ▼
                             ┌─────────┐
                             │  Open   │
                             └─────────┘
```

### 3. Bulkhead

Isolate failures to prevent cascade:

```
Without Bulkhead:
┌────────────────────────────────────────┐
│            Thread Pool (100)           │
│  All requests share the same pool      │
│  One slow service can exhaust all      │
└────────────────────────────────────────┘

With Bulkhead:
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Service A    │ │ Service B    │ │ Service C    │
│ Pool (30)    │ │ Pool (30)    │ │ Pool (40)    │
└──────────────┘ └──────────────┘ └──────────────┘
Slow Service A can only exhaust its own pool
```

### 4. Timeout

Always set timeouts on external calls:

```python
try:
    response = requests.get(url, timeout=5)  # 5 second timeout
except requests.Timeout:
    return fallback_response()
```

### 5. Fallback

Provide alternative when primary fails:

```python
def get_user_recommendations(user_id):
    try:
        return recommendation_service.get(user_id)
    except ServiceUnavailable:
        return get_popular_items()  # Fallback to popular items
```

### 6. Failover

Automatic switch to backup:

```
Primary                     Secondary
┌─────────────┐             ┌─────────────┐
│   Active    │────────────▶│   Standby   │
│   (writes)  │ replication │   (ready)   │
└─────────────┘             └─────────────┘
      │                           │
      ✗ Failure                   │
                                  ▼
                            ┌─────────────┐
                            │   Active    │
                            │   (writes)  │
                            └─────────────┘
```

## Key Takeaways

1. **Measure availability** using SLIs, set targets with SLOs
2. **Redundancy is key** - eliminate single points of failure
3. **Design for failure** - assume everything will fail
4. **Use error budgets** to balance reliability and velocity
5. **Defense in depth** - multiple fault tolerance patterns
6. **MTTR matters** - fast recovery beats rare failures

## Practice Questions

1. Calculate the availability of a system with three 99.9% services in series.
2. How would you design a system with 99.99% availability?
3. What's the difference between graceful degradation and failover?
4. How do you decide on the right SLO for a service?

## Further Reading

- [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/) - Google
- [Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard

---

Next: [Consistency Models](../03-consistency/README.md)
