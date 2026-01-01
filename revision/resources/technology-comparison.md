# Technology Comparison

When to use what technology.

## Databases

### SQL (PostgreSQL, MySQL)
```
Use when:
  ✓ Need transactions (ACID)
  ✓ Complex queries with JOINs
  ✓ Data is relational
  ✓ Schema is stable
  ✓ Strong consistency required

Avoid when:
  ✗ Massive write throughput needed
  ✗ Schema changes frequently
  ✗ Horizontal scaling is priority
```

### NoSQL: Document (MongoDB)
```
Use when:
  ✓ Flexible schema needed
  ✓ Document-like data (JSON)
  ✓ Rapid prototyping
  ✓ Embedded relationships

Avoid when:
  ✗ Many-to-many relationships
  ✗ Complex transactions
  ✗ Need JOINs across collections
```

### NoSQL: Wide-Column (Cassandra)
```
Use when:
  ✓ Massive write throughput
  ✓ Time-series data
  ✓ Known query patterns
  ✓ Geographic distribution

Avoid when:
  ✗ Need ad-hoc queries
  ✗ Complex aggregations
  ✗ Strong consistency required
```

### NoSQL: Key-Value (Redis, DynamoDB)
```
Use when:
  ✓ Simple get/put operations
  ✓ Caching
  ✓ Session storage
  ✓ Real-time data

Avoid when:
  ✗ Need complex queries
  ✗ Data relationships
  ✗ Large values (>1MB)
```

### Graph (Neo4j)
```
Use when:
  ✓ Many-to-many relationships
  ✓ Social networks
  ✓ Recommendation engines
  ✓ Path finding queries

Avoid when:
  ✗ Simple data access patterns
  ✗ Need high write throughput
  ✗ Large-scale analytics
```

## Message Queues

### Kafka
```
Use when:
  ✓ High throughput (millions/sec)
  ✓ Event streaming
  ✓ Need to replay messages
  ✓ Multiple consumers per message

Avoid when:
  ✗ Simple pub/sub
  ✗ Team is small
  ✗ Low latency is critical
```

### RabbitMQ
```
Use when:
  ✓ Complex routing needed
  ✓ Traditional queuing
  ✓ Priority queues
  ✓ Simpler operations

Avoid when:
  ✗ Massive scale
  ✗ Need message replay
  ✗ Long retention
```

### SQS
```
Use when:
  ✓ AWS ecosystem
  ✓ Simple queuing
  ✓ Managed service preferred
  ✓ Pay-per-use model

Avoid when:
  ✗ Need strict ordering
  ✗ Sub-second latency
  ✗ Multi-cloud
```

## Caching

### Redis
```
Use when:
  ✓ Need data structures (lists, sets, sorted sets)
  ✓ Pub/sub needed
  ✓ Atomic operations
  ✓ Persistence optional

Best for:
  - Session cache
  - Leaderboards
  - Rate limiting
  - Real-time analytics
```

### Memcached
```
Use when:
  ✓ Simple key-value caching
  ✓ Multi-threaded performance
  ✓ Memory efficiency

Best for:
  - Simple object caching
  - HTML fragment caching
```

### CDN (CloudFront, Cloudflare)
```
Use when:
  ✓ Static content (images, videos, JS, CSS)
  ✓ Global users
  ✓ Cacheable API responses

Avoid when:
  ✗ Personalized content
  ✗ Frequent updates
  ✗ Private data
```

## Search

### Elasticsearch
```
Use when:
  ✓ Full-text search
  ✓ Log analytics
  ✓ Faceted search
  ✓ Near real-time search

Avoid when:
  ✗ Primary database
  ✗ Strong consistency needed
  ✗ Simple queries (use SQL)
```

### Algolia
```
Use when:
  ✓ Need managed search
  ✓ Autocomplete/typeahead
  ✓ E-commerce search
  ✓ Mobile-first

Avoid when:
  ✗ Budget constraints
  ✗ Custom ranking
  ✗ Self-hosted required
```

## Storage

### S3 / Object Storage
```
Use when:
  ✓ Large files (images, videos)
  ✓ Backup/archive
  ✓ Static website hosting
  ✓ Unlimited scale

Avoid when:
  ✗ Small frequent updates
  ✗ Need file system semantics
  ✗ Low latency required
```

### Block Storage (EBS)
```
Use when:
  ✓ Database storage
  ✓ File systems
  ✓ Low latency I/O
  ✓ Frequently updated data
```

### HDFS / Data Lake
```
Use when:
  ✓ Big data analytics
  ✓ Batch processing
  ✓ Data warehouse
  ✓ Schema-on-read
```

## Load Balancing

### Application Load Balancer (L7)
```
Use when:
  ✓ HTTP/HTTPS traffic
  ✓ Path-based routing
  ✓ SSL termination
  ✓ WebSocket support
```

### Network Load Balancer (L4)
```
Use when:
  ✓ TCP/UDP traffic
  ✓ Extreme performance needed
  ✓ Static IP required
  ✓ Non-HTTP protocols
```

## Container Orchestration

### Kubernetes
```
Use when:
  ✓ Many microservices
  ✓ Need portability
  ✓ Complex deployment patterns
  ✓ Team has expertise

Avoid when:
  ✗ Small team
  ✗ Simple application
  ✗ Learning curve too steep
```

### ECS / Fargate
```
Use when:
  ✓ AWS ecosystem
  ✓ Simpler orchestration
  ✓ Managed service preferred
  ✓ Faster to learn
```

## Decision Matrix

### For Chat Application
```
Messages:     Cassandra (write-heavy)
User data:    PostgreSQL
Presence:     Redis
Notifications: Kafka
Search:       Elasticsearch
```

### For E-Commerce
```
Products:     PostgreSQL
Cart:         Redis
Orders:       PostgreSQL
Search:       Elasticsearch
Images:       S3 + CDN
Events:       Kafka
```

### For Social Media
```
Posts:        Cassandra (scale)
Users:        PostgreSQL
Feed:         Redis
Images:       S3 + CDN
Search:       Elasticsearch
Notifications: Kafka
```

### For Analytics Platform
```
Events:       Kafka → S3
Processing:   Spark
Warehouse:    Snowflake/BigQuery
Dashboard:    Redis (real-time)
```

---

[Back to Resources](./README.md)
