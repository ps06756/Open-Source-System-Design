# Search Engines

Search engines enable fast full-text search across large datasets. Understanding how they work is crucial for designing search functionality.

## Table of Contents
- [Why Search Engines?](#why-search-engines)
- [How Search Works](#how-search-works)
- [Elasticsearch](#elasticsearch)
- [Search Features](#search-features)
- [Design Considerations](#design-considerations)
- [Key Takeaways](#key-takeaways)

## Why Search Engines?

### Database Search Limitations

```sql
-- Database LIKE query
SELECT * FROM products WHERE name LIKE '%wireless headphones%';

Problems:
- Full table scan (slow)
- No relevance ranking
- No typo tolerance
- No synonyms ("headphones" vs "earphones")
```

### Search Engine Benefits

```
┌────────────────────────────────────────────────────────┐
│           Search Engine Advantages                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Speed: Inverted index → O(1) lookup                  │
│  Relevance: TF-IDF, BM25 scoring                      │
│  Typos: Fuzzy matching ("headphonse" → "headphones") │
│  Synonyms: "laptop" finds "notebook"                  │
│  Facets: Filter by category, price range             │
│  Suggestions: Autocomplete, "did you mean"           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## How Search Works

### Inverted Index

```
Documents:
Doc 1: "wireless bluetooth headphones"
Doc 2: "wired headphones with microphone"
Doc 3: "wireless bluetooth speaker"

Inverted Index:
┌─────────────┬─────────────┐
│    Term     │  Documents  │
├─────────────┼─────────────┤
│ wireless    │ [1, 3]      │
│ bluetooth   │ [1, 3]      │
│ headphones  │ [1, 2]      │
│ wired       │ [2]         │
│ microphone  │ [2]         │
│ speaker     │ [3]         │
└─────────────┴─────────────┘

Query: "wireless headphones"
→ wireless: [1, 3]
→ headphones: [1, 2]
→ Intersection: [1] (Doc 1 matches best)
```

### Text Analysis Pipeline

```
Original: "The Quick Brown Fox Jumps!"
           │
           ▼
┌─────────────────────────────────────────────────────────┐
│  1. Character Filter                                    │
│     Remove HTML, convert special chars                  │
│     → "The Quick Brown Fox Jumps"                      │
│                                                         │
│  2. Tokenizer                                           │
│     Split into words                                    │
│     → ["The", "Quick", "Brown", "Fox", "Jumps"]        │
│                                                         │
│  3. Token Filters                                       │
│     - Lowercase: ["the", "quick", "brown", "fox", "jumps"]
│     - Stop words: ["quick", "brown", "fox", "jumps"]   │
│     - Stemming: ["quick", "brown", "fox", "jump"]      │
└─────────────────────────────────────────────────────────┘
```

### Relevance Scoring

```
TF-IDF (Term Frequency - Inverse Document Frequency):

TF = times term appears in document / total terms in document
IDF = log(total documents / documents containing term)
Score = TF × IDF

Example:
- "wireless" appears 2 times in 100-word doc: TF = 0.02
- 100 docs total, 10 contain "wireless": IDF = log(10) = 1
- Score = 0.02 × 1 = 0.02
```

## Elasticsearch

Elasticsearch is the most popular open-source search engine, built on Apache Lucene.

### Architecture

```
┌────────────────────────────────────────────────────────┐
│                 Elasticsearch Cluster                  │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │                    Index                          │ │
│  │  ┌─────────────┐  ┌─────────────┐               │ │
│  │  │  Shard 0    │  │  Shard 1    │  ...          │ │
│  │  │ (Primary)   │  │ (Primary)   │               │ │
│  │  └──────┬──────┘  └──────┬──────┘               │ │
│  │         │                │                       │ │
│  │    ┌────┴────┐      ┌────┴────┐                 │ │
│  │    ▼         ▼      ▼         ▼                 │ │
│  │  Replica   Replica  Replica   Replica           │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  Node 1          Node 2          Node 3               │
└────────────────────────────────────────────────────────┘
```

### Index and Mapping

```json
// Create index with mapping
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "english"
      },
      "description": {
        "type": "text"
      },
      "price": {
        "type": "float"
      },
      "category": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date"
      }
    }
  }
}
```

### Indexing Documents

```json
// Index a document
POST /products/_doc
{
  "name": "Wireless Bluetooth Headphones",
  "description": "High-quality audio with noise cancellation",
  "price": 99.99,
  "category": "electronics",
  "created_at": "2024-01-15"
}
```

### Searching

```json
// Full-text search
GET /products/_search
{
  "query": {
    "match": {
      "name": "wireless headphones"
    }
  }
}

// Bool query (AND/OR/NOT)
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "headphones" } }
      ],
      "filter": [
        { "range": { "price": { "lte": 100 } } },
        { "term": { "category": "electronics" } }
      ]
    }
  }
}
```

## Search Features

### Autocomplete

```json
// Completion suggester
PUT /products
{
  "mappings": {
    "properties": {
      "name_suggest": {
        "type": "completion"
      }
    }
  }
}

// Query
GET /products/_search
{
  "suggest": {
    "product-suggest": {
      "prefix": "wire",
      "completion": {
        "field": "name_suggest"
      }
    }
  }
}

Result: ["wireless headphones", "wired earbuds", ...]
```

### Fuzzy Search

```json
// Handle typos
GET /products/_search
{
  "query": {
    "match": {
      "name": {
        "query": "headphonse",
        "fuzziness": "AUTO"
      }
    }
  }
}

// Finds "headphones" despite typo
```

### Faceted Search

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "aggs": {
    "categories": {
      "terms": { "field": "category" }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 50 },
          { "from": 50, "to": 100 },
          { "from": 100 }
        ]
      }
    }
  }
}

// Returns counts per category and price range
```

### Highlighting

```json
GET /products/_search
{
  "query": {
    "match": { "description": "noise cancellation" }
  },
  "highlight": {
    "fields": {
      "description": {}
    }
  }
}

// Result: "High-quality audio with <em>noise cancellation</em>"
```

## Design Considerations

### Indexing Strategy

```
┌────────────────────────────────────────────────────────┐
│              Sync Strategy                             │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Option 1: Synchronous                                 │
│  App ──▶ Database ──▶ Elasticsearch                   │
│  Pros: Immediate consistency                          │
│  Cons: Slower writes, coupling                        │
│                                                        │
│  Option 2: Async (Queue)                              │
│  App ──▶ Database                                     │
│      ──▶ Queue ──▶ Indexer ──▶ Elasticsearch         │
│  Pros: Decoupled, faster writes                       │
│  Cons: Eventually consistent                          │
│                                                        │
│  Option 3: Change Data Capture                        │
│  Database ──CDC──▶ Debezium ──▶ Elasticsearch        │
│  Pros: No app changes needed                          │
│  Cons: More infrastructure                            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Scaling

```
Read scaling: Add replicas
Write scaling: Add shards (must plan upfront)

┌────────────────────────────────────────────────────────┐
│  Sharding strategy:                                    │
│  - 1 shard handles ~20-40 GB                          │
│  - Too few: Write bottleneck                          │
│  - Too many: Overhead, slow queries                   │
│  - Can't change after creation!                       │
│                                                        │
│  Recommendation:                                       │
│  - 5-10 shards for small indices                     │
│  - 1 shard per 20GB for large indices                │
└────────────────────────────────────────────────────────┘
```

### Alternatives

| Engine | Strengths | Use Case |
|--------|-----------|----------|
| Elasticsearch | Feature-rich, ecosystem | General search |
| OpenSearch | Elasticsearch fork, AWS | AWS users |
| Algolia | SaaS, fast, easy | E-commerce, sites |
| Meilisearch | Simple, fast, typo-tolerant | Small-medium apps |
| Typesense | Fast, easy to use | Real-time search |

## Key Takeaways

1. **Inverted index** is the key data structure
2. **Text analysis** affects search quality
3. **Elasticsearch** is the industry standard
4. **Keep DB and search in sync** (async recommended)
5. **Plan sharding** upfront - can't change later
6. **Use specialized features** - autocomplete, fuzzy, facets

## Practice Questions

1. Design a product search for an e-commerce site.
2. How would you keep Elasticsearch in sync with your database?
3. What's the difference between `text` and `keyword` types?
4. How would you implement "did you mean" suggestions?

## Further Reading

- [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Relevant Search](https://www.manning.com/books/relevant-search)
- [Algolia Documentation](https://www.algolia.com/doc/)

---

Next: [API Gateway](../09-api-gateway/README.md)
