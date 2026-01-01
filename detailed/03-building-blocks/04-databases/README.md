# Databases

Choosing the right database is one of the most important decisions in system design. Different databases excel at different use cases.

## Table of Contents
1. [SQL vs NoSQL](#sql-vs-nosql)
2. [Database Types](#database-types)
3. [Indexing](#indexing)
4. [Database Selection Guide](#database-selection-guide)
5. [Interview Questions](#interview-questions)

---

## SQL vs NoSQL

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          SQL vs NoSQL                                       │
├────────────────────────────────────┬───────────────────────────────────────┤
│              SQL                   │              NoSQL                     │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Structured data (tables)           │ Flexible schema                        │
│ ACID transactions                  │ BASE (eventual consistency)            │
│ Complex queries (JOINs)            │ Simple queries                         │
│ Vertical scaling                   │ Horizontal scaling                     │
│ Schema-on-write                    │ Schema-on-read                         │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Best for:                          │ Best for:                              │
│ • Financial transactions           │ • High-volume writes                   │
│ • Complex relationships            │ • Flexible/evolving schema             │
│ • Reports and analytics            │ • Simple access patterns               │
│ • Data integrity critical          │ • Massive scale                        │
├────────────────────────────────────┼───────────────────────────────────────┤
│ Examples:                          │ Examples:                              │
│ PostgreSQL, MySQL, Oracle          │ MongoDB, Cassandra, DynamoDB          │
└────────────────────────────────────┴───────────────────────────────────────┘
```

---

## Database Types

```
┌─────────────────────────────────────────────────────────────┐
│                    Database Types                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Relational (SQL)                                         │
│     Tables with rows and columns                            │
│     PostgreSQL, MySQL, Oracle                               │
│     Use: Transactions, complex queries                      │
│                                                              │
│  2. Document                                                 │
│     JSON-like documents                                     │
│     MongoDB, CouchDB                                        │
│     Use: Flexible schema, nested data                       │
│     {                                                        │
│       "user": "john",                                       │
│       "posts": [{"title": "..."}]                          │
│     }                                                        │
│                                                              │
│  3. Key-Value                                                │
│     Simple key → value pairs                                │
│     Redis, DynamoDB, Memcached                              │
│     Use: Caching, sessions, simple lookups                  │
│     user:123 → {"name": "John"}                            │
│                                                              │
│  4. Wide-Column                                              │
│     Rows with dynamic columns                               │
│     Cassandra, HBase, ScyllaDB                              │
│     Use: Time-series, write-heavy, analytics               │
│                                                              │
│  5. Graph                                                    │
│     Nodes and relationships                                 │
│     Neo4j, Amazon Neptune                                   │
│     Use: Social networks, recommendations                   │
│     (User)--[FOLLOWS]-->(User)                             │
│                                                              │
│  6. Time-Series                                              │
│     Optimized for timestamped data                          │
│     InfluxDB, TimescaleDB                                   │
│     Use: Metrics, IoT, monitoring                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Indexing

```
┌─────────────────────────────────────────────────────────────┐
│                      Indexing                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without index: Full table scan O(n)                        │
│  With index: B-tree lookup O(log n)                         │
│                                                              │
│  B-Tree Index (most common):                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    [M]                               │   │
│  │                   /   \                              │   │
│  │               [D,H]   [R,W]                         │   │
│  │              /  |  \   /  |  \                      │   │
│  │           [A-C][E-G][I-L][N-Q][S-V][X-Z]            │   │
│  └─────────────────────────────────────────────────────┘   │
│  Good for: Range queries, sorting, equality               │
│                                                              │
│  Hash Index:                                                 │
│  hash(key) → location                                       │
│  Good for: Exact matches only                              │
│  Bad for: Range queries                                     │
│                                                              │
│  Types of Indexes:                                           │
│  • Primary: On primary key (unique, clustered)              │
│  • Secondary: On other columns                              │
│  • Composite: On multiple columns                           │
│  • Covering: Includes all query columns                     │
│                                                              │
│  Trade-offs:                                                 │
│  ✓ Faster reads                                             │
│  ✗ Slower writes (index must be updated)                   │
│  ✗ More storage                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Database Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│              Database Selection Guide                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Need transactions?                                          │
│  └── Yes → PostgreSQL / MySQL                               │
│                                                              │
│  Flexible schema?                                            │
│  └── Yes → MongoDB / DynamoDB                               │
│                                                              │
│  High write throughput?                                      │
│  └── Yes → Cassandra / ScyllaDB                             │
│                                                              │
│  Graph relationships?                                        │
│  └── Yes → Neo4j                                            │
│                                                              │
│  Caching / Sessions?                                         │
│  └── Yes → Redis                                            │
│                                                              │
│  Full-text search?                                           │
│  └── Yes → Elasticsearch                                    │
│                                                              │
│  Time-series data?                                           │
│  └── Yes → InfluxDB / TimescaleDB                          │
│                                                              │
│  Common Combinations:                                        │
│  • PostgreSQL + Redis (primary + cache)                    │
│  • MongoDB + Elasticsearch (storage + search)              │
│  • Cassandra + Redis (write-heavy + caching)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. When would you choose SQL vs NoSQL?
2. What is database indexing and why is it important?

### Intermediate
3. How would you design a database schema for a social media app?
4. Explain ACID properties.

### Advanced
5. How would you scale a PostgreSQL database to handle millions of users?

---

[← Previous: CDN](../03-cdn/README.md) | [Next: Caching →](../05-caching/README.md)
