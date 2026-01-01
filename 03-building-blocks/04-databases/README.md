# Databases

Understanding database options and their trade-offs is crucial for system design. This chapter covers SQL, NoSQL, and when to use each.

## Table of Contents
- [SQL Databases](#sql-databases)
- [NoSQL Databases](#nosql-databases)
- [SQL vs NoSQL](#sql-vs-nosql)
- [Database Indexing](#database-indexing)
- [Scaling Databases](#scaling-databases)
- [Key Takeaways](#key-takeaways)

## SQL Databases

Relational databases store data in tables with predefined schemas and support ACID transactions.

### ACID Properties

```
┌────────────────────────────────────────────────────────┐
│                    ACID                                │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Atomicity:     All or nothing                        │
│                 Transaction fully completes or fails   │
│                                                        │
│  Consistency:   Valid state to valid state            │
│                 Constraints always satisfied           │
│                                                        │
│  Isolation:     Transactions don't interfere          │
│                 Concurrent execution = serial          │
│                                                        │
│  Durability:    Committed = permanent                 │
│                 Survives crashes                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Common SQL Databases

| Database | Strengths | Use Case |
|----------|-----------|----------|
| PostgreSQL | Feature-rich, extensible | General purpose |
| MySQL | Fast reads, replication | Web applications |
| SQLite | Embedded, zero-config | Mobile, embedded |
| SQL Server | Enterprise, .NET | Enterprise apps |
| Oracle | Enterprise, mature | Large enterprises |

### Schema Example

```sql
-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table with foreign key
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    total DECIMAL(10, 2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for common queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
```

## NoSQL Databases

Non-relational databases optimized for specific data models and access patterns.

### Types of NoSQL

```
┌────────────────────────────────────────────────────────┐
│                  NoSQL Types                           │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Document Store                                        │
│  ├── MongoDB, CouchDB                                 │
│  └── JSON documents, flexible schema                  │
│                                                        │
│  Key-Value Store                                       │
│  ├── Redis, DynamoDB, Memcached                       │
│  └── Simple key → value, very fast                   │
│                                                        │
│  Wide-Column Store                                     │
│  ├── Cassandra, HBase, BigTable                       │
│  └── Columns grouped in families                      │
│                                                        │
│  Graph Database                                        │
│  ├── Neo4j, Amazon Neptune                            │
│  └── Nodes and relationships                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Document Store (MongoDB)

```javascript
// Flexible schema - documents in same collection can differ
{
  "_id": ObjectId("..."),
  "email": "user@example.com",
  "name": "John Doe",
  "orders": [
    {
      "id": "order-123",
      "items": [...],
      "total": 99.99
    }
  ],
  "preferences": {
    "newsletter": true,
    "theme": "dark"
  }
}
```

### Key-Value Store (Redis)

```redis
# Simple key-value
SET user:123:name "John Doe"
GET user:123:name

# Hash
HSET user:123 name "John Doe" email "john@example.com"
HGET user:123 name

# List
LPUSH notifications:123 "New message"
LRANGE notifications:123 0 10

# Set
SADD user:123:followers user:456 user:789
SMEMBERS user:123:followers
```

### Wide-Column Store (Cassandra)

```cql
-- Partition key + clustering columns
CREATE TABLE messages (
    chat_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    content TEXT,
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Efficient query: all messages in a chat
SELECT * FROM messages WHERE chat_id = ?;

-- Inefficient: requires scanning all partitions
SELECT * FROM messages WHERE sender_id = ?;
```

### Graph Database (Neo4j)

```cypher
// Create nodes
CREATE (john:Person {name: 'John'})
CREATE (jane:Person {name: 'Jane'})
CREATE (acme:Company {name: 'ACME'})

// Create relationships
CREATE (john)-[:WORKS_AT]->(acme)
CREATE (jane)-[:WORKS_AT]->(acme)
CREATE (john)-[:KNOWS]->(jane)

// Query: Find coworkers
MATCH (p:Person)-[:WORKS_AT]->(:Company)<-[:WORKS_AT]-(coworker:Person)
WHERE p.name = 'John'
RETURN coworker.name
```

## SQL vs NoSQL

### When to Use SQL

```
✓ Complex queries with JOINs
✓ Transactions across multiple tables
✓ Well-defined schema
✓ Data integrity is critical
✓ Moderate scale (millions of rows)

Examples:
- E-commerce orders
- Financial systems
- User accounts
- Inventory management
```

### When to Use NoSQL

```
✓ Simple query patterns
✓ Massive scale (billions of rows)
✓ Flexible/evolving schema
✓ High write throughput
✓ Specific data model needs

Examples:
- Session storage (key-value)
- User profiles (document)
- Time-series data (wide-column)
- Social networks (graph)
```

### Comparison

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| Schema | Rigid | Flexible |
| Scaling | Vertical (mostly) | Horizontal |
| Transactions | ACID | BASE (usually) |
| Queries | Complex JOINs | Simple lookups |
| Consistency | Strong | Eventual (often) |
| Use case | General purpose | Specific patterns |

## Database Indexing

### How Indexes Work

```
Without index (table scan):
┌─────┬──────────┬─────────┐
│ id  │  email   │  name   │  Scan all 1M rows
├─────┼──────────┼─────────┤  to find email
│  1  │  a@...   │  Alice  │
│  2  │  b@...   │  Bob    │
│ ... │  ...     │  ...    │
│ 1M  │  z@...   │  Zoe    │
└─────┴──────────┴─────────┘

With B-tree index on email:
                    ┌───────────────┐
                    │  m@example.com│
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
        ┌───────────┐               ┌───────────┐
        │  f@...    │               │  t@...    │
        └─────┬─────┘               └─────┬─────┘
              │                           │
    ┌─────────┴─────────┐       ┌─────────┴─────────┐
    ▼                   ▼       ▼                   ▼
┌───────┐         ┌───────┐ ┌───────┐         ┌───────┐
│ a-e   │         │ g-l   │ │ n-s   │         │ u-z   │
└───────┘         └───────┘ └───────┘         └───────┘

O(log n) lookup instead of O(n)
```

### Index Types

```
┌────────────────────────────────────────────────────────┐
│                   Index Types                          │
├────────────────────────────────────────────────────────┤
│                                                        │
│  B-tree (default)                                      │
│  - Balanced tree structure                            │
│  - Good for: =, <, >, BETWEEN, LIKE 'prefix%'        │
│                                                        │
│  Hash                                                  │
│  - Direct lookup                                       │
│  - Good for: = only                                   │
│  - O(1) vs O(log n)                                   │
│                                                        │
│  GIN (Generalized Inverted)                           │
│  - Multiple values per row                            │
│  - Good for: Arrays, full-text, JSONB                │
│                                                        │
│  GiST (Generalized Search Tree)                       │
│  - Good for: Geometric, range types                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Index Best Practices

```sql
-- Composite index (column order matters!)
CREATE INDEX idx_orders_user_status
ON orders(user_id, status);

-- Good: Uses full index
SELECT * FROM orders WHERE user_id = 1 AND status = 'pending';

-- Good: Uses left prefix
SELECT * FROM orders WHERE user_id = 1;

-- Bad: Can't use index (missing left column)
SELECT * FROM orders WHERE status = 'pending';

-- Covering index (includes all needed columns)
CREATE INDEX idx_orders_covering
ON orders(user_id, status) INCLUDE (total, created_at);

-- Query uses index only, no table lookup
SELECT total, created_at FROM orders
WHERE user_id = 1 AND status = 'pending';
```

### Index Trade-offs

```
Pros:
+ Faster reads (O(log n) vs O(n))
+ Enables efficient sorting
+ Supports unique constraints

Cons:
- Slower writes (must update index)
- Storage overhead
- More to maintain
```

## Scaling Databases

### Read Replicas

```
┌─────────────────────────────────────────────────────────┐
│                   Read Replicas                         │
│                                                         │
│              Writes                                     │
│                │                                        │
│                ▼                                        │
│         ┌───────────┐                                  │
│         │  Primary  │                                  │
│         └─────┬─────┘                                  │
│               │ Replication                            │
│    ┌──────────┼──────────┐                            │
│    ▼          ▼          ▼                            │
│ ┌──────┐  ┌──────┐  ┌──────┐                         │
│ │Replica│  │Replica│  │Replica│◀── Reads             │
│ └──────┘  └──────┘  └──────┘                         │
│                                                         │
│ Scale reads: Add more replicas                         │
│ Limitation: Write bottleneck remains                   │
└─────────────────────────────────────────────────────────┘
```

### Sharding

```
┌─────────────────────────────────────────────────────────┐
│                     Sharding                            │
│                                                         │
│              ┌─────────────────┐                       │
│              │    Router       │                       │
│              │  (shard key)    │                       │
│              └───────┬─────────┘                       │
│                      │                                  │
│        ┌─────────────┼─────────────┐                   │
│        ▼             ▼             ▼                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ Shard 1  │  │ Shard 2  │  │ Shard 3  │            │
│  │ (A-H)    │  │ (I-P)    │  │ (Q-Z)    │            │
│  └──────────┘  └──────────┘  └──────────┘            │
│                                                         │
│ Scale writes: Add more shards                          │
│ Limitation: Cross-shard queries                        │
└─────────────────────────────────────────────────────────┘
```

### Connection Pooling

```
Without pooling:
Request → New connection → Query → Close connection
           (30ms overhead per request)

With pooling (PgBouncer, ProxySQL):
Request → Get from pool → Query → Return to pool
           (0ms overhead)

┌────────────────────────────────────────────────────────┐
│  App Server                    Connection Pool         │
│  ┌──────────┐                  ┌────────────┐         │
│  │ Request 1│──────────────────│ Conn 1     │         │
│  │ Request 2│──────────────────│ Conn 2     │──▶ DB  │
│  │ Request 3│──────────────────│ Conn 3     │         │
│  │ ...      │                  │ ...        │         │
│  └──────────┘                  └────────────┘         │
│                                                        │
│  1000 requests → 10 connections (multiplexed)         │
└────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **SQL for complex queries** and transactions
2. **NoSQL for scale** and specific data models
3. **Index strategically** - not too few, not too many
4. **Read replicas** for read scaling
5. **Sharding** for write scaling (with complexity)
6. **Connection pooling** is essential at scale

## Practice Questions

1. Design a database schema for a social media application.
2. When would you choose MongoDB over PostgreSQL?
3. How would you optimize a slow query?
4. Design a sharding strategy for a user database.

## Further Reading

- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

Next: [Caching](../05-caching/README.md)
