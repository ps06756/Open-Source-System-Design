# Scalability

Scalability is the ability of a system to handle increased load by adding resources. This is perhaps the most fundamental concept in system design.

## Table of Contents
- [What is Scalability?](#what-is-scalability)
- [Vertical Scaling](#vertical-scaling-scale-up)
- [Horizontal Scaling](#horizontal-scaling-scale-out)
- [Comparison](#comparison)
- [Scaling Strategies](#scaling-strategies)
- [Database Scaling](#database-scaling)
- [Key Takeaways](#key-takeaways)

## What is Scalability?

A system is scalable if it can efficiently accommodate growth in:
- **Users**: More concurrent users
- **Data**: Larger datasets
- **Traffic**: Higher request rates
- **Complexity**: More features

```
                    Load
                      │
                      │        ╱ Scalable system
            Response  │      ╱
              Time    │    ╱
                      │  ╱
                      │╱__________ Non-scalable system
                      └──────────────────────▶
                              Users/Requests
```

## Vertical Scaling (Scale Up)

Add more power to existing machines.

```
Before:                          After:
┌─────────────────┐              ┌─────────────────┐
│    Server       │              │    Server       │
│                 │              │                 │
│  CPU: 4 cores   │    ──▶      │  CPU: 32 cores  │
│  RAM: 16 GB     │              │  RAM: 256 GB    │
│  Disk: 500 GB   │              │  Disk: 4 TB SSD │
└─────────────────┘              └─────────────────┘
```

### Advantages

| Advantage | Description |
|-----------|-------------|
| Simplicity | No code changes required |
| No distributed complexity | Single machine, no network issues |
| Strong consistency | ACID transactions work naturally |
| Lower latency | No network hops |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Hardware limits | Can't scale beyond biggest machine |
| Cost | High-end hardware is expensive |
| Single point of failure | One machine = one failure domain |
| Downtime | Upgrades require restarts |

### When to Use

- Early-stage startups (simpler)
- Small to medium workloads
- Workloads requiring strong consistency
- When horizontal scaling is complex (legacy systems)

## Horizontal Scaling (Scale Out)

Add more machines to the pool.

```
Before:                          After:
┌─────────────┐                  ┌─────────────┐
│   Server    │                  │   Server 1  │
└─────────────┘                  └─────────────┘
                                 ┌─────────────┐
                      ──▶       │   Server 2  │
                                 └─────────────┘
                                 ┌─────────────┐
                                 │   Server 3  │
                                 └─────────────┘
                                 ┌─────────────┐
                                 │   Server N  │
                                 └─────────────┘
```

### With Load Balancer

```
              ┌─────────────────┐
              │  Load Balancer  │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
         ▼             ▼             ▼
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Server 1 │  │ Server 2 │  │ Server 3 │
   └──────────┘  └──────────┘  └──────────┘
```

### Advantages

| Advantage | Description |
|-----------|-------------|
| Unlimited scaling | Add more commodity machines |
| Cost effective | Cheaper hardware at scale |
| Fault tolerance | Survive individual failures |
| No downtime | Rolling deployments |
| Geographic distribution | Deploy closer to users |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Complexity | Distributed systems are hard |
| Consistency challenges | CAP theorem trade-offs |
| Operational overhead | More machines to manage |
| Code changes | Applications must be stateless |

### Requirements for Horizontal Scaling

1. **Stateless applications**: Store state externally
2. **Load balancing**: Distribute traffic evenly
3. **Service discovery**: Find available instances
4. **Data partitioning**: Distribute data across nodes

## Comparison

| Aspect | Vertical | Horizontal |
|--------|----------|------------|
| Complexity | Lower | Higher |
| Cost curve | Exponential | Linear |
| Downtime for scaling | Required | Not required |
| Max capacity | Limited | Unlimited |
| Failure impact | Total | Partial |
| Data consistency | Easy | Hard |

### Cost Comparison

```
Cost
  │
  │                              ╱ Vertical
  │                            ╱
  │                          ╱
  │                        ╱
  │              ╱────────
  │            ╱  Horizontal
  │          ╱
  │        ╱
  │      ╱
  │    ╱
  │  ╱
  └──────────────────────────────▶
                              Capacity
```

## Scaling Strategies

### 1. Stateless Services

Move state out of application servers:

```
Before (Stateful):
┌─────────────┐
│   Server    │
│  ┌───────┐  │
│  │Session│  │  ◀── Can't scale horizontally
│  │ Data  │  │
│  └───────┘  │
└─────────────┘

After (Stateless):
┌─────────────┐     ┌─────────────┐
│   Server    │────▶│    Redis    │
│ (stateless) │     │  (sessions) │
└─────────────┘     └─────────────┘
```

### 2. Load Balancing Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| Round Robin | Rotate through servers | Equal capacity servers |
| Weighted Round Robin | Proportional to weights | Different capacity servers |
| Least Connections | Send to least busy | Long-lived connections |
| IP Hash | Same IP to same server | Session affinity |
| Random | Random selection | Simple, low overhead |

### 3. Auto-scaling

```
                                         ┌─────────────┐
┌───────────┐     ┌──────────────┐       │   Server    │
│  Metrics  │────▶│  Auto-scaler │──────▶│   Server    │
│ (CPU,Mem) │     │   (rules)    │       │   Server    │
└───────────┘     └──────────────┘       │   Server    │
                                         └─────────────┘
                                              ▲
                                              │
                                         Add/Remove
                                         based on rules
```

**Scaling policies:**
- **Target tracking**: Maintain 70% CPU
- **Step scaling**: Add 2 instances if CPU > 80%
- **Scheduled**: Scale up at 9 AM, down at 6 PM

### 4. Caching

Reduce load on backend systems:

```
┌────────┐     ┌─────────┐     ┌──────────┐
│ Client │────▶│  Cache  │────▶│ Database │
└────────┘     └─────────┘     └──────────┘
                   │
                   │ Cache hit = fast
                   │ Cache miss = slow + populate cache
```

### 5. Asynchronous Processing

Move work out of the request path:

```
Synchronous:
Client ──▶ API ──▶ Process ──▶ Database ──▶ Response (slow)

Asynchronous:
Client ──▶ API ──▶ Queue ──▶ Response (fast)
                     │
                     └──▶ Worker ──▶ Database (background)
```

## Database Scaling

### Read Replicas

Scale reads horizontally:

```
                    Writes
                      │
                      ▼
              ┌───────────────┐
              │    Primary    │
              └───────┬───────┘
                      │ Replication
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Replica  │  │ Replica  │  │ Replica  │
  └──────────┘  └──────────┘  └──────────┘
        ▲             ▲             ▲
        └─────────────┼─────────────┘
                      │
                    Reads
```

### Sharding

Scale writes horizontally:

```
                    ┌──────────────┐
                    │   Router     │
                    │ (shard key)  │
                    └──────┬───────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐      ┌──────────┐      ┌──────────┐
   │ Shard 1  │      │ Shard 2  │      │ Shard 3  │
   │ (A-H)    │      │ (I-P)    │      │ (Q-Z)    │
   └──────────┘      └──────────┘      └──────────┘
```

### Scaling Strategies by Layer

| Layer | Strategy |
|-------|----------|
| Web/API | Horizontal scaling with load balancer |
| Application | Stateless services, auto-scaling |
| Cache | Distributed cache (Redis Cluster) |
| Database | Read replicas, sharding |
| Storage | Object storage (S3), CDN |

## Key Takeaways

1. **Start vertical, go horizontal** when you hit limits
2. **Make services stateless** to enable horizontal scaling
3. **Scale reads first** (replicas), then writes (sharding)
4. **Use caching** to reduce load on databases
5. **Async processing** for non-critical work
6. **Auto-scale** based on metrics, not manually

## Practice Questions

1. Your service handles 1000 RPS. How would you handle 100,000 RPS?
2. What makes an application stateless? Give examples of state.
3. When would you choose vertical over horizontal scaling?
4. How do you handle user sessions with horizontal scaling?

## Further Reading

- [Scalability for Dummies](https://www.lecloud.net/tagged/scalability)
- [The Art of Scalability](https://www.amazon.com/Art-Scalability-Scalable-Architecture-Organizations/dp/0134032802)

---

Next: [Availability & Reliability](../02-availability/README.md)
