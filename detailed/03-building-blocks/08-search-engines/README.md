# Search Engines

Search engines enable fast full-text search across large datasets. Understanding inverted indexes and search architecture is crucial for many system designs.

## Table of Contents
1. [Inverted Index](#inverted-index)
2. [Elasticsearch Architecture](#elasticsearch-architecture)
3. [Search vs Database](#search-vs-database)
4. [Interview Questions](#interview-questions)

---

## Inverted Index

```
┌─────────────────────────────────────────────────────────────┐
│                    Inverted Index                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Documents:                                                  │
│  Doc 1: "The quick brown fox"                               │
│  Doc 2: "The quick dog"                                     │
│  Doc 3: "The brown dog jumps"                               │
│                                                              │
│  Inverted Index:                                             │
│  ┌────────────┬────────────────────────────────┐            │
│  │   Term     │   Document IDs                  │            │
│  ├────────────┼────────────────────────────────┤            │
│  │   the      │   [1, 2, 3]                    │            │
│  │   quick    │   [1, 2]                       │            │
│  │   brown    │   [1, 3]                       │            │
│  │   fox      │   [1]                          │            │
│  │   dog      │   [2, 3]                       │            │
│  │   jumps    │   [3]                          │            │
│  └────────────┴────────────────────────────────┘            │
│                                                              │
│  Search "quick brown":                                       │
│  quick → [1, 2]                                             │
│  brown → [1, 3]                                             │
│  Intersection → [1]  ✓ Doc 1 matches!                       │
│                                                              │
│  Why it's fast:                                              │
│  • O(1) lookup per term                                     │
│  • Set operations for combining                             │
│  • Pre-computed at index time                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Elasticsearch Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               Elasticsearch Cluster                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                     Cluster                          │   │
│  │                                                      │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │ Node 1  │  │ Node 2  │  │ Node 3  │             │   │
│  │  │ (Master)│  │         │  │         │             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  │       │            │            │                   │   │
│  │  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐            │   │
│  │  │ Shard 0 │  │ Shard 1 │  │ Shard 2 │            │   │
│  │  │ (P)     │  │ (P)     │  │ (P)     │            │   │
│  │  │ Shard 1 │  │ Shard 2 │  │ Shard 0 │            │   │
│  │  │ (R)     │  │ (R)     │  │ (R)     │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  │                                                      │   │
│  │  P = Primary shard, R = Replica shard               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  Index = Collection of documents (like a DB table)          │
│  Shard = Partition of an index                              │
│  Replica = Copy of a shard for HA                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Search vs Database

| Aspect | Search Engine | Database |
|--------|--------------|----------|
| **Primary Use** | Full-text search | Data storage |
| **Query Type** | Text matching, relevance | Exact matches, SQL |
| **Consistency** | Eventually consistent | Strong (usually) |
| **Updates** | Near real-time | Immediate |
| **Best For** | Search, analytics | Source of truth |

**Pattern**: Database (primary) + Search Engine (secondary)
- Write to DB first, then sync to search
- Read from search for queries
- Read from DB for exact lookups

---

## Interview Questions

### Basic
1. What is an inverted index?

### Intermediate
2. How would you implement search for an e-commerce site?

### Advanced
3. Design a search system that handles millions of documents.

---

[← Previous: Blob Storage](../07-blob-storage/README.md) | [Next: API Gateway →](../09-api-gateway/README.md)
