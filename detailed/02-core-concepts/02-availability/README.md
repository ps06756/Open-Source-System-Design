# Availability & Reliability

Availability measures how often a system is operational and accessible. Reliability measures how long a system can perform without failure. Both are critical for production systems.

## Table of Contents
1. [Measuring Availability](#measuring-availability)
2. [SLAs, SLOs, and SLIs](#slas-slos-and-slis)
3. [Fault Tolerance](#fault-tolerance)
4. [Redundancy Patterns](#redundancy-patterns)
5. [Failure Modes](#failure-modes)
6. [Interview Questions](#interview-questions)

---

## Measuring Availability

### The Nines of Availability

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         Availability Levels                                 │
├──────────────┬────────────────┬─────────────────┬──────────────────────────┤
│ Availability │ Downtime/Year  │ Downtime/Month  │ Example                  │
├──────────────┼────────────────┼─────────────────┼──────────────────────────┤
│ 99%          │ 3.65 days      │ 7.31 hours      │ Internal tools           │
│ 99.9%        │ 8.77 hours     │ 43.83 minutes   │ Business apps            │
│ 99.95%       │ 4.38 hours     │ 21.92 minutes   │ E-commerce               │
│ 99.99%       │ 52.60 minutes  │ 4.38 minutes    │ Payment systems          │
│ 99.999%      │ 5.26 minutes   │ 26.30 seconds   │ Critical infrastructure  │
│ 99.9999%     │ 31.56 seconds  │ 2.63 seconds    │ Telecom, hospitals       │
└──────────────┴────────────────┴─────────────────┴──────────────────────────┘
```

### Calculating Availability

```
┌─────────────────────────────────────────────────────────────┐
│                 Availability Formulas                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Single Component:                                           │
│                                                              │
│            MTBF                                              │
│  A = ─────────────────                                      │
│       MTBF + MTTR                                            │
│                                                              │
│  MTBF = Mean Time Between Failures                          │
│  MTTR = Mean Time To Recovery                               │
│                                                              │
│  Example:                                                    │
│  MTBF = 1000 hours, MTTR = 1 hour                           │
│  A = 1000 / (1000 + 1) = 99.9%                              │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Serial Components (both must work):                        │
│                                                              │
│  A_total = A₁ × A₂ × A₃ × ... × Aₙ                         │
│                                                              │
│  Example: Web → App → DB (each 99.9%)                       │
│  A_total = 0.999 × 0.999 × 0.999 = 99.7%                   │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Parallel Components (one must work):                       │
│                                                              │
│  A_total = 1 - (1 - A₁) × (1 - A₂) × ... × (1 - Aₙ)       │
│                                                              │
│  Example: Two servers (each 99%)                            │
│  A_total = 1 - (0.01 × 0.01) = 99.99%                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## SLAs, SLOs, and SLIs

### Definitions

```
┌─────────────────────────────────────────────────────────────┐
│                  SLA, SLO, SLI Hierarchy                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                        SLA                             │  │
│  │           Service Level Agreement                      │  │
│  │    (Contract with customers, includes penalties)       │  │
│  │                                                        │  │
│  │  Example: "99.9% uptime or 10% credit"                │  │
│  │                                                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │                      SLO                         │  │  │
│  │  │           Service Level Objective               │  │  │
│  │  │     (Internal target, stricter than SLA)        │  │  │
│  │  │                                                  │  │  │
│  │  │  Example: "99.95% uptime (internal goal)"       │  │  │
│  │  │                                                  │  │  │
│  │  │  ┌───────────────────────────────────────────┐  │  │  │
│  │  │  │                   SLI                      │  │  │  │
│  │  │  │        Service Level Indicator            │  │  │  │
│  │  │  │      (Actual measurement/metric)          │  │  │  │
│  │  │  │                                           │  │  │  │
│  │  │  │  Example: "99.97% requests succeeded"     │  │  │  │
│  │  │  └───────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Common SLIs

| Category | SLI | Target |
|----------|-----|--------|
| **Availability** | Success rate of requests | 99.9% |
| **Latency** | p99 response time | < 200ms |
| **Throughput** | Requests per second | > 10,000 |
| **Error Rate** | Failed requests / Total | < 0.1% |
| **Durability** | Data loss probability | 99.999999999% |

### Error Budgets

```
┌─────────────────────────────────────────────────────────────┐
│                     Error Budget                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SLO: 99.9% availability                                     │
│  Error Budget: 0.1% = 43.8 minutes/month                    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Month Progress: ████████████░░░░░░░░ (60%)         │    │
│  │  Error Budget:   █████░░░░░░░░░░░░░░░ (25% used)    │    │
│  │                                                      │    │
│  │  Budget remaining: 32.8 minutes                     │    │
│  │  Days remaining: 12                                  │    │
│  │  Safe to deploy: ✓ Yes                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Rules:                                                      │
│  • Budget remaining → Can deploy new features               │
│  • Budget exhausted → Focus on reliability                  │
│  • Balance innovation with stability                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Fault Tolerance

### Strategies

```
┌─────────────────────────────────────────────────────────────┐
│              Fault Tolerance Strategies                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Redundancy                                               │
│     Multiple instances of components                         │
│     Example: 3 replicas of each service                     │
│                                                              │
│  2. Failover                                                 │
│     Switch to backup when primary fails                     │
│     Example: Primary → Secondary database                   │
│                                                              │
│  3. Graceful Degradation                                     │
│     Reduce functionality instead of complete failure        │
│     Example: Show cached data when DB is down               │
│                                                              │
│  4. Circuit Breaker                                          │
│     Stop calling failing services                           │
│     Example: Disable payment service, queue orders          │
│                                                              │
│  5. Bulkhead                                                 │
│     Isolate failures to prevent cascade                     │
│     Example: Separate thread pools per service              │
│                                                              │
│  6. Retry with Backoff                                       │
│     Retry failed operations with increasing delay           │
│     Example: 1s → 2s → 4s → 8s → fail                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Health Checks

```
┌─────────────────────────────────────────────────────────────┐
│                     Health Checks                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Types:                                                      │
│                                                              │
│  1. Liveness Check                                           │
│     "Is the process running?"                               │
│     GET /healthz → 200 OK                                   │
│     Failed → Restart container                              │
│                                                              │
│  2. Readiness Check                                          │
│     "Can it handle requests?"                               │
│     GET /ready → 200 OK                                     │
│     Failed → Remove from load balancer                      │
│                                                              │
│  3. Startup Check                                            │
│     "Has it finished initializing?"                         │
│     Prevents liveness checks during startup                 │
│                                                              │
│  Example response:                                           │
│  {                                                           │
│    "status": "healthy",                                     │
│    "checks": {                                              │
│      "database": "up",                                      │
│      "redis": "up",                                         │
│      "disk": "ok (80% free)"                               │
│    }                                                         │
│  }                                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Redundancy Patterns

### Active-Passive (Failover)

```
┌─────────────────────────────────────────────────────────────┐
│                    Active-Passive                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Normal operation:                                           │
│  ┌─────────────────┐     ┌─────────────────┐               │
│  │     Active      │     │    Passive      │               │
│  │   (Serving)     │     │  (Standby)      │               │
│  │       ●         │ ──▶ │       ○         │               │
│  │   Traffic       │     │   No Traffic    │               │
│  └─────────────────┘     └─────────────────┘               │
│                                                              │
│  After failure:                                              │
│  ┌─────────────────┐     ┌─────────────────┐               │
│  │     Failed      │     │     Active      │               │
│  │       ✗         │     │   (Promoted)    │               │
│  │   No Traffic    │     │       ●         │               │
│  └─────────────────┘     └─────────────────┘               │
│                                                              │
│  Pros: Simple, consistent state                             │
│  Cons: Resource waste (passive idle), failover time         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Active-Active

```
┌─────────────────────────────────────────────────────────────┐
│                     Active-Active                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Normal operation:                                           │
│           ┌─────────────────┐                               │
│           │  Load Balancer  │                               │
│           └────────┬────────┘                               │
│        ┌───────────┴───────────┐                            │
│        │                       │                            │
│        ▼                       ▼                            │
│  ┌──────────────┐       ┌──────────────┐                   │
│  │   Active 1   │       │   Active 2   │                   │
│  │  (50% load)  │       │  (50% load)  │                   │
│  │      ●       │       │      ●       │                   │
│  └──────────────┘       └──────────────┘                   │
│                                                              │
│  After one failure:                                          │
│  ┌──────────────┐       ┌──────────────┐                   │
│  │    Failed    │       │   Active 2   │                   │
│  │      ✗       │       │ (100% load)  │                   │
│  └──────────────┘       └──────────────┘                   │
│                                                              │
│  Pros: Full resource utilization, instant failover          │
│  Cons: Complex state sync, split-brain risk                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### N+1 Redundancy

```
┌─────────────────────────────────────────────────────────────┐
│                     N+1 Redundancy                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Capacity needed: 3 servers at 100% each                    │
│  Deploy: 4 servers at 75% each                              │
│                                                              │
│  Normal:                                                     │
│  [75%] [75%] [75%] [75%]  ← All 4 active                   │
│                                                              │
│  One fails:                                                  │
│  [100%] [100%] [100%] [✗]  ← Still have full capacity      │
│                                                              │
│  For critical systems: N+2 or 2N                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Failure Modes

### Types of Failures

```
┌─────────────────────────────────────────────────────────────┐
│                    Failure Types                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Crash Failure                                            │
│     Component stops working completely                       │
│     Detectable, can failover                                │
│                                                              │
│  2. Omission Failure                                         │
│     Component fails to respond (timeout)                    │
│     Network issues, overload                                │
│                                                              │
│  3. Timing Failure                                           │
│     Response too slow (SLA violation)                       │
│     Performance degradation                                  │
│                                                              │
│  4. Byzantine Failure                                        │
│     Component behaves arbitrarily wrong                     │
│     Hardest to detect, may return wrong data                │
│                                                              │
│  5. Cascade Failure                                          │
│     One failure triggers others                             │
│     Example: Overload → Timeout → Retry storm              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Preventing Cascade Failures

```
┌─────────────────────────────────────────────────────────────┐
│              Preventing Cascade Failures                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  The Cascade:                                                │
│  Service A slow → B times out → B retries →                │
│  A more overloaded → More timeouts → System collapse        │
│                                                              │
│  Prevention:                                                 │
│                                                              │
│  1. Timeouts                                                 │
│     Set aggressive timeouts, fail fast                      │
│     Don't wait forever for slow service                     │
│                                                              │
│  2. Circuit Breakers                                         │
│     ┌──────────┐  failures  ┌──────────┐                   │
│     │  Closed  │ ─────────▶ │   Open   │                   │
│     │ (normal) │            │ (reject) │                   │
│     └──────────┘            └────┬─────┘                   │
│          ▲                       │ timeout                  │
│          │      ┌────────────────┘                          │
│          │      ▼                                           │
│          │ ┌──────────┐                                     │
│          └─│Half-Open │ (test one request)                  │
│            └──────────┘                                     │
│                                                              │
│  3. Rate Limiting                                            │
│     Limit requests to protect services                      │
│                                                              │
│  4. Load Shedding                                            │
│     Reject requests when overloaded                         │
│     Return 503 early vs timeout                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the difference between availability and reliability?
2. Calculate the availability of a system with 99.9% uptime.
3. What are SLAs, SLOs, and SLIs?

### Intermediate
4. How would you design a system with 99.99% availability?
5. Explain active-passive vs active-active failover.
6. What is an error budget and how is it used?

### Advanced
7. How do you prevent cascade failures in microservices?
8. Design a health check system for a distributed application.
9. How would you handle a situation where you're running low on error budget?

---

[← Previous: Scalability](../01-scalability/README.md) | [Next: Consistency →](../03-consistency/README.md)
