# Patterns Cheat Sheet

Common patterns for system design problems.

## Scaling Patterns

### Horizontal Scaling
```
When: Need more capacity
How:  Add more machines behind load balancer
Example: Web servers, stateless services
```

### Vertical Scaling
```
When: Quick fix, stateful services
How:  Upgrade to bigger machine
Limit: Hardware limits, single point of failure
```

### Database Sharding
```
When: Single DB can't handle load
How:  Split data across multiple DBs
Strategies:
  - By user_id (hash or range)
  - By geography
  - By time (for time-series data)
```

### Read Replicas
```
When: Read-heavy workload
How:  Replicate writes to read-only copies
Trade-off: Eventual consistency on reads
```

## Caching Patterns

### Cache-Aside (Lazy Loading)
```
Read: Check cache → if miss, read DB, update cache
Write: Update DB, invalidate cache
Use: General purpose, simple
```

### Write-Through
```
Read: Check cache → if miss, read DB
Write: Update cache AND DB together
Use: When reads follow writes
```

### Write-Behind (Write-Back)
```
Read: From cache
Write: To cache only, async persist to DB
Use: High write throughput
Risk: Data loss if cache fails
```

### Cache Invalidation Strategies
```
TTL:        Expire after time (simple but may be stale)
Event:      Invalidate on data change (complex but fresh)
Versioning: Include version in cache key
```

## Data Patterns

### Event Sourcing
```
Instead of storing current state:
  Store sequence of events
  Rebuild state by replaying events
Benefits:
  - Complete audit trail
  - Time travel debugging
  - Event replay for analytics
```

### CQRS
```
Separate read and write models:
  Command: Optimized for writes
  Query:   Optimized for reads (denormalized)
Use when read/write patterns differ significantly
```

### Saga Pattern
```
For distributed transactions:
  1. Execute step 1
  2. If step 2 fails, compensate step 1
  3. Each step has a rollback action
Use: Multi-service transactions (e-commerce order)
```

## Communication Patterns

### Request-Response (Sync)
```
HTTP/REST, gRPC
Simple, direct
Blocking, coupling
```

### Message Queue (Async)
```
Kafka, RabbitMQ, SQS
Decoupling, resilience
Complexity, eventual consistency
```

### Pub/Sub
```
Publishers don't know subscribers
Use: Notifications, events, fan-out
Example: Redis Pub/Sub, Kafka topics
```

### Long Polling
```
Client: Opens connection
Server: Holds until data or timeout
Use: Near real-time without WebSocket
```

### WebSocket
```
Full duplex connection
Use: Chat, gaming, live updates
Overhead: Connection management
```

## Reliability Patterns

### Circuit Breaker
```
States: Closed → Open → Half-Open
When service fails repeatedly:
  1. Stop calling (Open)
  2. Wait and test (Half-Open)
  3. Resume if healthy (Closed)
```

### Retry with Backoff
```
On failure:
  Wait 1s, retry
  Wait 2s, retry
  Wait 4s, retry
  Give up
Add jitter to prevent thundering herd
```

### Bulkhead
```
Isolate failures:
  - Separate thread pools per service
  - Separate connection pools
  - Service mesh sidecars
```

### Rate Limiting
```
Algorithms:
  Token Bucket: Allows bursts
  Leaky Bucket: Smooth rate
  Fixed Window: Simple but edge cases
  Sliding Window: Most accurate
```

## Consistency Patterns

### Strong Consistency
```
All reads see latest write
How: Synchronous replication, consensus
Trade-off: Higher latency, lower availability
Use: Financial transactions
```

### Eventual Consistency
```
Reads may see stale data temporarily
How: Async replication
Trade-off: Stale reads possible
Use: Social media, analytics
```

### Read-Your-Writes
```
User sees their own writes immediately
Others may see stale data
How: Route reads to write primary, or sticky sessions
```

## Search Patterns

### Inverted Index
```
word → [doc1, doc2, doc3]
Fast full-text search
Used by: Elasticsearch, Lucene
```

### Trie
```
Prefix tree for autocomplete
O(k) lookup where k = key length
Memory intensive
```

### Bloom Filter
```
Probabilistic "is in set?" check
False positives possible
False negatives impossible
Use: Cache miss optimization
```

## ID Generation Patterns

### UUID
```
Random, 128-bit
No coordination needed
Not sortable, large size
```

### Snowflake ID
```
64-bit: timestamp + machine ID + sequence
Time-ordered, unique, distributed
Requires clock sync
```

### Database Sequence
```
Simple auto-increment
Requires coordination
Single point of failure
```

## Storage Patterns

### LSM Tree
```
Log-Structured Merge Tree
Write-optimized
Memtable → SSTables → Compaction
Used by: Cassandra, RocksDB, LevelDB
```

### B-Tree
```
Read-optimized
In-place updates
Used by: PostgreSQL, MySQL (InnoDB)
```

### Time-Series Optimization
```
Partition by time
Compress older data
TTL for expiration
Used by: InfluxDB, TimescaleDB
```

## API Patterns

### Pagination
```
Offset-based: ?page=2&limit=20 (simple but slow for large offsets)
Cursor-based: ?cursor=abc123 (efficient, no skipped items)
```

### API Versioning
```
URL: /api/v1/users
Header: Accept: application/vnd.api+json; version=1
Query: /api/users?version=1
```

### Idempotency
```
Same request → same result
How: Idempotency key in request
Store: Request hash → result
```

---

[Back to Resources](./README.md)
