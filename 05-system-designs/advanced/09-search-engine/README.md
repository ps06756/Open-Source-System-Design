# Design a Search Engine

Design a web search engine like Google.

## 1. Requirements

### Functional Requirements
- Crawl and index web pages
- Process search queries
- Return ranked results
- Handle billions of documents
- Autocomplete suggestions
- Spell correction

### Non-Functional Requirements
- Low latency (<500ms for queries)
- High availability
- Fresh index (hours to days old)
- Relevance and quality results
- Scale: 100B+ pages, 100K QPS

### Capacity Estimation

```
Pages: 100 billion
Average page size: 100 KB (text extracted)
Index size: 100B × 100 KB = 10 PB (raw)
Inverted index: ~1 PB (compressed)

Queries: 100K/sec
Average query: 3 words
Index lookups: 300K/sec
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Crawling Pipeline                      │ │
│  │  URL Frontier → Fetcher → Parser → Content Store        │ │
│  └────────────────────────────┬─────────────────────────────┘ │
│                               │                                │
│                               ▼                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   Indexing Pipeline                       │ │
│  │  Tokenizer → Analyzer → Inverted Index Builder          │ │
│  └────────────────────────────┬─────────────────────────────┘ │
│                               │                                │
│                               ▼                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                      Index Servers                        │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │ │
│  │  │ Shard 1 │ │ Shard 2 │ │ Shard 3 │ │ Shard N │       │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘       │ │
│  └────────────────────────────┬─────────────────────────────┘ │
│                               │                                │
│                               ▼                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                    Query Pipeline                         │ │
│  │  Query Parser → Search → Ranking → Result Assembly      │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Inverted Index

```
┌────────────────────────────────────────────────────────────────┐
│                    Inverted Index                              │
│                                                                │
│  Forward index: doc_id → [words]                              │
│  Inverted index: word → [doc_ids]                             │
│                                                                │
│  Example:                                                      │
│  Doc 1: "the quick brown fox"                                 │
│  Doc 2: "the quick dog"                                       │
│  Doc 3: "brown fox jumps"                                     │
│                                                                │
│  Inverted Index:                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  "the"    → [(1, [0]), (2, [0])]                        │  │
│  │  "quick"  → [(1, [1]), (2, [1])]                        │  │
│  │  "brown"  → [(1, [2]), (3, [0])]                        │  │
│  │  "fox"    → [(1, [3]), (3, [1])]                        │  │
│  │  "dog"    → [(2, [2])]                                   │  │
│  │  "jumps"  → [(3, [2])]                                   │  │
│  │                                                          │  │
│  │  Format: word → [(doc_id, [positions]), ...]            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Postings list includes:                                      │
│  - Document ID                                                │
│  - Term frequency in document                                 │
│  - Positions (for phrase queries)                            │
│  - Additional signals (title, anchor text)                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Index Building

```python
class IndexBuilder:
    def __init__(self):
        self.inverted_index = {}
        self.doc_store = {}

    def add_document(self, doc_id, content, metadata):
        """Index a document"""
        # Store document
        self.doc_store[doc_id] = {
            'url': metadata['url'],
            'title': metadata['title'],
            'snippet': content[:500]
        }

        # Tokenize and normalize
        tokens = self.tokenize(content)

        # Build term frequencies
        term_freq = {}
        for position, token in enumerate(tokens):
            if token not in term_freq:
                term_freq[token] = {'count': 0, 'positions': []}
            term_freq[token]['count'] += 1
            term_freq[token]['positions'].append(position)

        # Add to inverted index
        for term, data in term_freq.items():
            if term not in self.inverted_index:
                self.inverted_index[term] = []

            self.inverted_index[term].append({
                'doc_id': doc_id,
                'frequency': data['count'],
                'positions': data['positions'],
                'in_title': term in self.tokenize(metadata['title'].lower())
            })

    def tokenize(self, text):
        """Tokenize and normalize text"""
        # Lowercase
        text = text.lower()

        # Remove punctuation
        text = re.sub(r'[^\w\s]', ' ', text)

        # Split into tokens
        tokens = text.split()

        # Remove stop words
        tokens = [t for t in tokens if t not in STOP_WORDS]

        # Stem/lemmatize
        tokens = [self.stemmer.stem(t) for t in tokens]

        return tokens
```

## 4. Index Sharding

```
┌────────────────────────────────────────────────────────────────┐
│                    Index Sharding                              │
│                                                                │
│  Two approaches:                                               │
│                                                                │
│  1. Document Partitioning                                     │
│     - Documents distributed across shards                    │
│     - Each shard has full vocabulary                         │
│     - Query goes to ALL shards, results merged               │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Shard 1: docs 1-1M    (full index for these docs)     │  │
│  │  Shard 2: docs 1M-2M   (full index for these docs)     │  │
│  │  Shard 3: docs 2M-3M   (full index for these docs)     │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  2. Term Partitioning                                         │
│     - Terms distributed across shards                        │
│     - Query term determines which shard to query             │
│     - Less common due to complexity                          │
│                                                                │
│  Google uses document partitioning with replication          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 5. Query Processing

```python
class QueryProcessor:
    def search(self, query, limit=10):
        """Process search query"""
        # Parse query
        parsed = self.parse_query(query)

        # Search all shards in parallel
        shard_results = await self.search_shards(parsed)

        # Merge and rank results
        candidates = self.merge_results(shard_results)

        # Apply final ranking
        ranked = self.rank(parsed, candidates)

        return ranked[:limit]

    def parse_query(self, query):
        """Parse and analyze query"""
        # Tokenize
        tokens = self.tokenize(query)

        # Detect special operators
        operators = {
            'site:': self.extract_site_filter,
            '"': self.extract_phrase,
            '-': self.extract_exclusion
        }

        parsed = {
            'terms': tokens,
            'phrases': [],
            'exclusions': [],
            'site_filter': None
        }

        # Apply spell correction
        corrected_terms = []
        for term in tokens:
            if not self.is_in_vocabulary(term):
                suggestion = self.spell_correct(term)
                corrected_terms.append(suggestion)
            else:
                corrected_terms.append(term)

        parsed['corrected_terms'] = corrected_terms

        return parsed

    async def search_shard(self, shard_id, parsed):
        """Search a single shard"""
        results = []

        # Get postings for each term
        term_postings = {}
        for term in parsed['terms']:
            postings = await self.index[shard_id].get_postings(term)
            term_postings[term] = postings

        # Intersect for AND queries
        if len(term_postings) > 1:
            doc_ids = self.intersect_postings(term_postings.values())
        else:
            doc_ids = list(term_postings.values())[0]

        # Calculate relevance scores
        for doc_id in doc_ids:
            score = self.calculate_score(doc_id, parsed, term_postings)
            results.append({'doc_id': doc_id, 'score': score})

        return results
```

## 6. Ranking Algorithm

```
┌────────────────────────────────────────────────────────────────┐
│                    Ranking (Simplified)                        │
│                                                                │
│  Score = TextRelevance × PageQuality × Freshness             │
│                                                                │
│  TextRelevance (TF-IDF / BM25):                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  TF = term frequency in document                        │  │
│  │  IDF = log(N / df)  where df = docs containing term    │  │
│  │                                                          │  │
│  │  BM25 (improved TF-IDF):                                │  │
│  │  score = IDF × (TF × (k1 + 1)) / (TF + k1 × (1 - b + b ×│  │
│  │          docLength/avgDocLength))                       │  │
│  │                                                          │  │
│  │  k1 = 1.2, b = 0.75 (typical values)                   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  PageQuality (PageRank):                                      │
│  - Based on link graph                                        │
│  - Pages with more inbound links from quality pages rank higher│
│                                                                │
│  Other signals:                                                │
│  - Term in title/URL                                         │
│  - Anchor text of inbound links                              │
│  - User engagement (click-through rate)                      │
│  - Page speed, mobile-friendliness                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### BM25 Implementation

```python
class BM25Scorer:
    def __init__(self, k1=1.2, b=0.75):
        self.k1 = k1
        self.b = b

    def score(self, query_terms, doc_id, index):
        """Calculate BM25 score for document"""
        score = 0
        doc_length = index.get_doc_length(doc_id)
        avg_doc_length = index.get_avg_doc_length()

        for term in query_terms:
            # Term frequency in document
            tf = index.get_term_frequency(term, doc_id)

            # Document frequency (number of docs containing term)
            df = index.get_document_frequency(term)

            # Inverse document frequency
            N = index.get_total_docs()
            idf = math.log((N - df + 0.5) / (df + 0.5) + 1)

            # BM25 formula
            numerator = tf * (self.k1 + 1)
            denominator = tf + self.k1 * (1 - self.b + self.b * doc_length / avg_doc_length)

            score += idf * (numerator / denominator)

        return score

class PageRank:
    def calculate(self, graph, damping=0.85, iterations=100):
        """Calculate PageRank for all pages"""
        N = len(graph)
        ranks = {page: 1/N for page in graph}

        for _ in range(iterations):
            new_ranks = {}
            for page in graph:
                rank = (1 - damping) / N

                for linking_page in graph.get_inbound_links(page):
                    outbound_count = len(graph.get_outbound_links(linking_page))
                    rank += damping * ranks[linking_page] / outbound_count

                new_ranks[page] = rank

            ranks = new_ranks

        return ranks
```

## 7. Query Optimization

```
┌────────────────────────────────────────────────────────────────┐
│                 Query Optimization                             │
│                                                                │
│  1. Term ordering: Query rare terms first                    │
│     - Reduces candidate set quickly                          │
│                                                                │
│  2. Early termination:                                        │
│     - Stop when enough high-quality results found            │
│     - Use tiered index (high-quality docs first)            │
│                                                                │
│  3. Caching:                                                   │
│     - Cache popular query results                            │
│     - Cache postings for common terms                        │
│                                                                │
│  4. Posting list compression:                                 │
│     - Delta encoding: [1, 5, 8] → [1, 4, 3]                 │
│     - Variable byte encoding                                  │
│     - Block compression                                       │
│                                                                │
│  5. Skip lists:                                               │
│     - Fast intersection of posting lists                     │
│     - Skip to doc_id without scanning                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. Key Takeaways

1. **Inverted index** - core data structure for fast lookup
2. **Document sharding** - distribute index across machines
3. **BM25** - improved TF-IDF for text relevance
4. **PageRank** - link-based quality signal
5. **Caching** - essential for performance
6. **Early termination** - optimize for common case

---

Next: [Ad Click Aggregator](../10-ad-click-aggregator/README.md)
