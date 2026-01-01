# Trade-offs Discussion

Every design decision has trade-offs. Discussing them shows maturity and depth.

## Why Trade-offs Matter

- There's no perfect solution
- Shows you understand alternatives
- Demonstrates engineering judgment
- Often the most important part of the interview

## Common Trade-off Dimensions

### 1. Consistency vs Availability
```
Strong Consistency:
- All reads see the most recent write
- May be unavailable during network partitions
- Example: Bank account balance

Eventual Consistency:
- Reads may see stale data temporarily
- Higher availability
- Example: Social media likes count
```

### 2. Latency vs Throughput
```
Optimize for Latency:
- Respond to individual requests quickly
- May process fewer total requests
- Example: Real-time gaming

Optimize for Throughput:
- Process many requests efficiently
- Individual requests may wait
- Example: Batch data processing
```

### 3. Read vs Write Performance
```
Read-optimized:
- Denormalize data
- Pre-compute results
- More storage, slower writes
- Example: News feed

Write-optimized:
- Normalize data
- Compute on read
- Less storage, faster writes
- Example: Analytics ingestion
```

### 4. Storage vs Compute
```
Store more (trade storage for compute):
- Pre-compute and cache results
- Faster reads
- Higher storage costs
- Example: Materialized views

Compute more (trade compute for storage):
- Calculate on demand
- Less storage
- Higher compute costs
- Example: Ad-hoc queries
```

### 5. Complexity vs Simplicity
```
Complex solution:
- More features
- Better performance
- Harder to maintain
- More failure points

Simple solution:
- Fewer features
- May need optimization later
- Easier to understand
- More reliable
```

## How to Discuss Trade-offs

### Framework

```
"For [decision], I considered [options A, B, C].

I chose [A] because:
- [Reason 1]
- [Reason 2]

The trade-off is:
- We lose [X]
- But we gain [Y]

If [condition changes], we might reconsider."
```

### Example: Database Choice

```
"For our database, I considered PostgreSQL, MongoDB, and Cassandra.

I chose PostgreSQL because:
- We need transactions for orders
- Our data is highly relational
- Team has SQL expertise

The trade-off is:
- We lose horizontal scalability of NoSQL
- But we gain ACID guarantees and query flexibility

If we hit scaling limits, we could shard or add a
read replica layer."
```

### Example: Caching Strategy

```
"For caching, I considered write-through, write-behind,
and cache-aside.

I chose cache-aside because:
- Simpler to implement
- Application controls what's cached
- Works well with our read-heavy workload

The trade-off is:
- We might have cache misses on first read
- But we avoid cache inconsistency issues

For hot data, we could add write-through to
eliminate cold starts."
```

## Trade-off Matrix

For important decisions, create a mental matrix:

```
Decision: Message Queue Technology

                   | Kafka    | RabbitMQ | SQS      |
-------------------|----------|----------|----------|
Throughput         | High     | Medium   | Medium   |
Latency            | Higher   | Low      | Medium   |
Ordering           | Partition| Queue    | FIFO opt |
Ops complexity     | High     | Medium   | Low      |
Cost               | High     | Medium   | Pay/use  |
Replay capability  | Yes      | No       | Limited  |

Choice: Kafka
Reason: We need high throughput and replay for
our analytics pipeline. Team can handle ops complexity.
```

## Common Trade-off Questions

### "Why not use X instead?"

Good response:
```
"X is a valid option. We could use X if [condition].
I chose Y because [reason specific to our requirements].
If [something changes], X might become preferable."
```

### "What would you do differently at 10× scale?"

Good response:
```
"At 10× scale, the bottleneck would likely be [component].
I'd consider:
1. [Optimization 1]
2. [Optimization 2]
3. [Architectural change]

The trade-off is increased complexity, but it's
necessary at that scale."
```

### "What are the failure modes?"

Good response:
```
"The main failure modes are:
1. [Service A] fails: Impact is [X], we mitigate by [Y]
2. [Database] fails: We have [replicas/failover]
3. [Network partition]: We've chosen [consistency/availability]

We accept [trade-off] to maintain [priority]."
```

## Real-World Trade-off Examples

### Twitter: Fan-out Strategy
```
Trade-off: Write latency vs Read latency

Decision: Hybrid fan-out
- Push to regular users (fast reads)
- Pull from celebrities (avoid write amplification)

Accept: Slightly more complex read path
Gain: Avoid 30-second delays when celebrity posts
```

### Netflix: Microservices
```
Trade-off: Simplicity vs Scalability

Decision: Microservices architecture
- Each service scales independently
- Teams own their services

Accept: Operational complexity, network calls
Gain: Can scale specific bottlenecks, faster deployment
```

### Amazon: DynamoDB
```
Trade-off: Flexibility vs Performance

Decision: Key-value with limited query patterns
- Design access patterns upfront
- Limited to specific queries

Accept: No ad-hoc queries, careful schema design
Gain: Consistent low-latency at any scale
```

## Tips for Trade-off Discussions

1. **Be explicit**: Don't assume the trade-off is obvious
2. **Be balanced**: Acknowledge both sides
3. **Be decisive**: Make a choice, don't waffle
4. **Be flexible**: Show willingness to reconsider
5. **Be practical**: Consider real-world constraints (team, time, cost)

---

Next: [Common Mistakes](../07-common-mistakes/README.md)
