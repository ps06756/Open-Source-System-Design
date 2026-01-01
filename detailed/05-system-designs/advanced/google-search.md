# Design Google Search

## 1. Requirements

### Functional Requirements
- Web crawling to discover pages
- Indexing of web content
- Search queries with ranked results
- Autocomplete suggestions
- Spell correction

### Non-Functional Requirements
- Low latency (< 500ms)
- High availability
- Fresh index (near real-time for news)
- Handle trillions of pages
- 100K+ queries per second

### Scale Estimates
- 100B+ web pages indexed
- 8.5B searches per day
- Average page: 10 KB
- Index size: Petabytes

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    Web Crawler                         │  │
│  │        (Discover and fetch web pages)                  │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │               Document Processing                      │  │
│  │    (Parse, extract text, detect language)             │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                     Indexer                            │  │
│  │         (Build inverted index)                         │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │               Distributed Index                        │  │
│  │          (Sharded across servers)                      │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                 Query Processor                        │  │
│  │      (Parse query, search index, rank results)        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Web Crawler

```
┌─────────────────────────────────────────────────────────────┐
│                      Web Crawler                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Crawler Architecture:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌─────────┐   ┌───────────┐   ┌────────────────────┐ │ │
│  │  │  Seed   │──▶│  URL      │──▶│   URL Frontier     │ │ │
│  │  │  URLs   │   │  Filter   │   │   (Priority Queue) │ │ │
│  │  └─────────┘   └───────────┘   └─────────┬──────────┘ │ │
│  │                                          │             │ │
│  │                    ┌─────────────────────┘             │ │
│  │                    ▼                                   │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │               Fetcher Cluster                    │  │ │
│  │  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐           │  │ │
│  │  │  │Fetch │ │Fetch │ │Fetch │ │Fetch │           │  │ │
│  │  │  │ er 1 │ │ er 2 │ │ er 3 │ │ er n │           │  │ │
│  │  │  └──────┘ └──────┘ └──────┘ └──────┘           │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                    │                                   │ │
│  │                    ▼                                   │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │            Document Store                        │  │ │
│  │  │      (Raw HTML for processing)                  │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Politeness:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Respect robots.txt                                  │ │
│  │  • Rate limit per domain (1 request per second)       │ │
│  │  • Identify as Googlebot (User-Agent)                 │ │
│  │  • Handle crawl-delay directives                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  URL Frontier Prioritization:                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Priority factors:                                     │ │
│  │  • Domain authority (PageRank)                        │ │
│  │  • Page freshness (news sites → more frequent)       │ │
│  │  • Content change frequency                           │ │
│  │  • Depth from seed URL                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Indexing

```
┌─────────────────────────────────────────────────────────────┐
│                       Indexing                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Inverted Index Structure:                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Forward Index (document → terms):                     │ │
│  │  Doc1: "the quick brown fox"                          │ │
│  │  Doc2: "the lazy dog"                                 │ │
│  │                                                         │ │
│  │  Inverted Index (term → documents):                    │ │
│  │  ┌────────────┬───────────────────────────────────┐   │ │
│  │  │   Term     │   Posting List                     │   │ │
│  │  ├────────────┼───────────────────────────────────┤   │ │
│  │  │   the      │ [Doc1:pos1, Doc2:pos1]            │   │ │
│  │  │   quick    │ [Doc1:pos2]                       │   │ │
│  │  │   brown    │ [Doc1:pos3]                       │   │ │
│  │  │   fox      │ [Doc1:pos4]                       │   │ │
│  │  │   lazy     │ [Doc2:pos2]                       │   │ │
│  │  │   dog      │ [Doc2:pos3]                       │   │ │
│  │  └────────────┴───────────────────────────────────┘   │ │
│  │                                                         │ │
│  │  Posting list includes:                                │ │
│  │  • Document ID                                         │ │
│  │  • Term frequency in document                         │ │
│  │  • Positions (for phrase queries)                     │ │
│  │  • Field (title, body, URL)                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Index Partitioning:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Option 1: Document-partitioned                        │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                  │ │
│  │  │Shard 1  │ │Shard 2  │ │Shard 3  │                  │ │
│  │  │Doc 1-1M │ │Doc 1M-2M│ │Doc 2M-3M│                  │ │
│  │  └─────────┘ └─────────┘ └─────────┘                  │ │
│  │  ✓ Each shard is self-contained                       │ │
│  │  ✓ Query hits all shards in parallel                  │ │
│  │                                                         │ │
│  │  Option 2: Term-partitioned                            │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                  │ │
│  │  │Shard 1  │ │Shard 2  │ │Shard 3  │                  │ │
│  │  │Terms A-H│ │Terms I-P│ │Terms Q-Z│                  │ │
│  │  └─────────┘ └─────────┘ └─────────┘                  │ │
│  │  ✓ Single shard per term                              │ │
│  │  ✗ Multi-term queries need coordination               │ │
│  │                                                         │ │
│  │  Google uses: Document-partitioned                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Query Processing & Ranking

```
┌─────────────────────────────────────────────────────────────┐
│              Query Processing & Ranking                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Query Flow:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Query parsing                                      │ │
│  │     "best pizza in NYC" →                              │ │
│  │     [best, pizza, in, NYC] + intent: local            │ │
│  │                                                         │ │
│  │  2. Query expansion                                    │ │
│  │     NYC → "New York City"                             │ │
│  │     pizza → "pizzeria", "italian restaurant"          │ │
│  │                                                         │ │
│  │  3. Index lookup (parallel across shards)             │ │
│  │     Each shard returns top-K candidates               │ │
│  │                                                         │ │
│  │  4. Merge and rank                                     │ │
│  │     Combine shard results, apply ranking model        │ │
│  │                                                         │ │
│  │  5. Return top 10 results                             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Ranking Signals (200+ factors):                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Content relevance:                                    │ │
│  │  • TF-IDF (term frequency × inverse doc frequency)   │ │
│  │  • BM25 (improved TF-IDF)                             │ │
│  │  • Title/URL match                                    │ │
│  │  • Phrase matches                                      │ │
│  │                                                         │ │
│  │  Page quality:                                         │ │
│  │  • PageRank (link analysis)                           │ │
│  │  • Domain authority                                    │ │
│  │  • Content freshness                                   │ │
│  │  • Page speed                                          │ │
│  │  • Mobile friendliness                                 │ │
│  │                                                         │ │
│  │  User signals:                                         │ │
│  │  • Click-through rate                                 │ │
│  │  • Dwell time                                         │ │
│  │  • Bounce rate                                         │ │
│  │  • Search history/personalization                     │ │
│  │                                                         │ │
│  │  Modern: BERT/Transformer models for understanding   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. PageRank

```
┌─────────────────────────────────────────────────────────────┐
│                      PageRank                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Core idea: Important pages are linked by important pages  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │     ┌───┐        ┌───┐                                 │ │
│  │     │ A │───────▶│ B │                                 │ │
│  │     └───┘        └─┬─┘                                 │ │
│  │       │            │                                    │ │
│  │       ▼            ▼                                    │ │
│  │     ┌───┐        ┌───┐                                 │ │
│  │     │ C │◀───────│ D │                                 │ │
│  │     └───┘        └───┘                                 │ │
│  │                                                         │ │
│  │  PR(page) = (1-d)/N + d × Σ(PR(linking_page) / outlinks)│
│  │                                                         │ │
│  │  d = damping factor (0.85)                             │ │
│  │  N = total pages                                        │ │
│  │                                                         │ │
│  │  Iterative computation until convergence               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Computing at scale:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • MapReduce/Spark for distributed computation        │ │
│  │  • Pre-computed and stored                            │ │
│  │  • Updated periodically (not real-time)               │ │
│  │  • Combined with other signals at query time          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Crawler** | Distributed, politeness-aware | Scale + respect |
| **Index** | Inverted, document-sharded | Fast lookups |
| **Storage** | Custom (Bigtable-like) | Optimized for workload |
| **Ranking** | ML + PageRank + 200 signals | Quality results |
| **Query** | Parallel shard queries | Low latency |

---

[Back to System Designs](../README.md)
