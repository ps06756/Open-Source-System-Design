# Design Typeahead / Autocomplete

Design a typeahead suggestion system like Google Search's autocomplete.

## 1. Requirements

### Functional Requirements
- Return top suggestions as user types
- Suggestions ranked by popularity/relevance
- Handle typos and fuzzy matching
- Personalized suggestions based on user history
- Support multiple languages

### Non-Functional Requirements
- Ultra-low latency (<100ms)
- High availability
- Handle 100K QPS
- Update suggestions based on trending queries

### Capacity Estimation

```
Queries: 10B searches/day
Typeahead requests: ~5 per search = 50B/day ≈ 600K/sec
Peak: 1M requests/sec

Unique queries: 5B (stored suggestions)
Average query: 20 characters
Storage: 5B × 20 bytes = 100 GB (just queries)
With metadata: ~500 GB
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│  Typeahead Service │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│                    ┌────────────────────────────┼───────────┐ │
│                    │                            │           │ │
│                    ▼                            ▼           │ │
│             ┌─────────────┐           ┌─────────────────┐   │ │
│             │  Trie Cache │           │  Personalization│   │ │
│             │   (Redis)   │           │     Service     │   │ │
│             └─────────────┘           └─────────────────┘   │ │
│                                                              │ │
│  ┌──────────────────────────────────────────────────────┐   │ │
│  │            Analytics Pipeline (Kafka + Spark)         │   │ │
│  │  (Updates suggestion rankings based on search data)  │   │ │
│  └──────────────────────────────────────────────────────┘   │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Data Structure: Trie

```
┌────────────────────────────────────────────────────────┐
│                      Trie Structure                    │
│                                                        │
│                        (root)                          │
│                       /      \                         │
│                      h        w                        │
│                     /          \                       │
│                    e            e                      │
│                   /              \                     │
│                  l                a                    │
│                 /                  \                   │
│                l ──▶ [hello: 100]  t                  │
│               /                     \                  │
│              o ──▶ [hello: 100]     h                 │
│                                      \                │
│                                       e               │
│                                        \              │
│                                         r ──▶ [weather: 80]│
│                                                        │
│  Each node stores:                                    │
│  - Character                                          │
│  - Is end of word?                                   │
│  - Top K suggestions for this prefix                 │
│  - Count/weight for ranking                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Trie Implementation

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_word = False
        self.weight = 0
        self.top_suggestions = []  # Pre-computed top K

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word, weight):
        node = self.root
        for char in word.lower():
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
        node.is_word = True
        node.weight = weight

    def get_suggestions(self, prefix, k=10):
        node = self.root

        # Navigate to prefix node
        for char in prefix.lower():
            if char not in node.children:
                return []
            node = node.children[char]

        # Return pre-computed suggestions
        return node.top_suggestions[:k]

    def precompute_suggestions(self, node, prefix="", k=10):
        """Pre-compute top K suggestions for each node"""
        suggestions = []

        if node.is_word:
            suggestions.append((prefix, node.weight))

        for char, child in node.children.items():
            child_suggestions = self.precompute_suggestions(
                child, prefix + char, k
            )
            suggestions.extend(child_suggestions)

        # Sort by weight and keep top K
        suggestions.sort(key=lambda x: -x[1])
        node.top_suggestions = suggestions[:k]

        return suggestions
```

## 4. Optimization: Prefix Hash Map

```
┌────────────────────────────────────────────────────────┐
│              Prefix Hash Map (for speed)              │
│                                                        │
│  Instead of traversing trie at runtime,              │
│  pre-compute all prefix → suggestions mappings       │
│                                                        │
│  Redis Hash:                                           │
│  Key: typeahead:prefix:{prefix}                       │
│  Value: JSON array of suggestions                    │
│                                                        │
│  "h"     → ["hello", "how", "help", "home", ...]     │
│  "he"    → ["hello", "help", "here", "health", ...]  │
│  "hel"   → ["hello", "help", "helmet", ...]          │
│  "hell"  → ["hello", "hell", ...]                    │
│  "hello" → ["hello", "hello world", ...]             │
│                                                        │
│  Storage: Limit to prefixes of length 1-10           │
│  For longer prefixes, use trie traversal            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Redis Implementation

```python
class TypeaheadService:
    def __init__(self, redis):
        self.redis = redis
        self.max_prefix_length = 10

    def get_suggestions(self, prefix, limit=10):
        prefix = prefix.lower().strip()

        if len(prefix) > self.max_prefix_length:
            # Fall back to trie for long prefixes
            return self.trie_lookup(prefix, limit)

        # O(1) lookup from Redis
        key = f"typeahead:{prefix}"
        suggestions = self.redis.get(key)

        if suggestions:
            return json.loads(suggestions)[:limit]

        return []

    def update_suggestions(self, prefix, suggestions):
        """Called by analytics pipeline"""
        key = f"typeahead:{prefix}"
        self.redis.set(key, json.dumps(suggestions))
        self.redis.expire(key, 86400)  # 24 hour TTL
```

## 5. Ranking Algorithm

```python
class SuggestionRanker:
    def rank(self, prefix, candidates, user_id=None):
        """
        Ranking factors:
        1. Query frequency (how often searched)
        2. Recency (trending now)
        3. User history (personalization)
        4. Edit distance (typo tolerance)
        """
        scored = []

        for candidate in candidates:
            score = 0

            # Base popularity score
            score += self.get_frequency_score(candidate)

            # Trending boost
            score += self.get_trending_boost(candidate) * 0.3

            # Personalization
            if user_id:
                score += self.get_personal_score(user_id, candidate) * 0.2

            # Exact prefix match boost
            if candidate.lower().startswith(prefix.lower()):
                score *= 1.5

            # Edit distance penalty for fuzzy matches
            distance = self.edit_distance(prefix, candidate[:len(prefix)])
            score *= (1 / (1 + distance))

            scored.append((candidate, score))

        scored.sort(key=lambda x: -x[1])
        return [s[0] for s in scored]

    def get_frequency_score(self, query):
        """Logarithmic scale to prevent runaway popular queries"""
        count = self.get_search_count(query)
        return math.log(count + 1)

    def get_trending_boost(self, query):
        """Compare recent frequency to historical average"""
        recent = self.get_recent_count(query, hours=24)
        historical = self.get_average_count(query)

        if historical == 0:
            return 0

        return min((recent / historical) - 1, 5)  # Cap at 5x boost
```

## 6. Analytics Pipeline

```
┌────────────────────────────────────────────────────────┐
│              Suggestion Update Pipeline                │
│                                                        │
│  Search logs → Kafka → Spark Streaming → Redis        │
│                                                        │
│  1. Collect search queries                            │
│     - Query text                                       │
│     - Timestamp                                        │
│     - User ID (for personalization)                   │
│     - Click-through (did user click result?)          │
│                                                        │
│  2. Aggregate counts                                   │
│     - Count queries per hour/day                      │
│     - Calculate moving averages                       │
│     - Detect trending queries                         │
│                                                        │
│  3. Update suggestions                                 │
│     - Rebuild trie periodically (hourly)             │
│     - Update Redis prefix mappings                   │
│     - Expire old/low-frequency queries               │
│                                                        │
│  4. Filter inappropriate content                      │
│     - Blocklist offensive terms                      │
│     - Remove spam queries                            │
│     - Comply with legal requirements                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Spark Job

```python
def update_suggestions_job():
    # Read search logs from Kafka
    df = spark.readStream \
        .format("kafka") \
        .option("subscribe", "search-queries") \
        .load()

    # Aggregate by prefix
    prefix_counts = df \
        .withColumn("prefixes", generate_prefixes(col("query"))) \
        .explode("prefixes") \
        .groupBy("prefix") \
        .agg(
            count("*").alias("count"),
            collect_list("query").alias("queries")
        )

    # Get top suggestions per prefix
    top_suggestions = prefix_counts \
        .withColumn("top_queries", get_top_k(col("queries"), 10))

    # Write to Redis
    top_suggestions.foreachPartition(write_to_redis)

def generate_prefixes(query):
    """Generate all prefixes of a query"""
    prefixes = []
    for i in range(1, min(len(query) + 1, 11)):
        prefixes.append(query[:i].lower())
    return prefixes
```

## 7. Personalization

```python
class PersonalizedTypeahead:
    def __init__(self, redis):
        self.redis = redis

    def get_suggestions(self, user_id, prefix, limit=10):
        # Get global suggestions
        global_suggestions = self.get_global_suggestions(prefix)

        # Get user's recent searches
        user_history = self.get_user_history(user_id, prefix)

        # Merge with user history boosted
        merged = self.merge_suggestions(
            global_suggestions,
            user_history,
            user_weight=0.3
        )

        return merged[:limit]

    def get_user_history(self, user_id, prefix):
        """Get user's past searches matching prefix"""
        key = f"user_searches:{user_id}"

        # Stored as sorted set with timestamps
        searches = self.redis.zrevrange(key, 0, 100)

        # Filter by prefix
        matching = [s for s in searches if s.startswith(prefix)]

        return matching[:10]

    def record_search(self, user_id, query):
        """Record user's search for personalization"""
        key = f"user_searches:{user_id}"
        self.redis.zadd(key, {query: time.time()})
        self.redis.zremrangebyrank(key, 0, -1001)  # Keep last 1000
```

## 8. Fuzzy Matching

```
┌────────────────────────────────────────────────────────┐
│                 Typo Tolerance                         │
│                                                        │
│  User types: "amazn"                                  │
│  Should suggest: "amazon"                             │
│                                                        │
│  Techniques:                                           │
│                                                        │
│  1. Edit Distance (Levenshtein)                       │
│     - Allow 1-2 character differences                │
│     - "amazn" → "amazon" (distance = 1)              │
│                                                        │
│  2. Phonetic Matching (Soundex/Metaphone)            │
│     - "john" and "jon" sound similar                 │
│     - Group by phonetic hash                         │
│                                                        │
│  3. N-gram Index                                       │
│     - "amazon" → ["am", "ma", "az", "zo", "on"]     │
│     - Find suggestions with similar n-grams          │
│                                                        │
│  4. Common Typo Patterns                              │
│     - Swapped adjacent keys: "amazno" → "amazon"    │
│     - Double letters: "ammazon" → "amazon"           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### BK-Tree for Fuzzy Search

```python
class BKTree:
    """Burkhard-Keller tree for efficient fuzzy search"""

    def __init__(self, distance_func):
        self.root = None
        self.distance = distance_func

    def insert(self, word):
        if self.root is None:
            self.root = BKNode(word)
            return

        node = self.root
        d = self.distance(word, node.word)

        while d in node.children:
            node = node.children[d]
            d = self.distance(word, node.word)

        node.children[d] = BKNode(word)

    def search(self, query, max_distance):
        """Find all words within edit distance"""
        results = []

        def _search(node):
            d = self.distance(query, node.word)

            if d <= max_distance:
                results.append((node.word, d))

            # Only traverse children within range
            for i in range(d - max_distance, d + max_distance + 1):
                if i in node.children:
                    _search(node.children[i])

        if self.root:
            _search(self.root)

        return results
```

## 9. Caching Strategy

```
┌────────────────────────────────────────────────────────┐
│              Multi-Level Caching                       │
│                                                        │
│  Level 1: Browser Cache                               │
│  - Cache suggestions for recently typed prefixes     │
│  - Reduces network requests                          │
│  - TTL: 5 minutes                                     │
│                                                        │
│  Level 2: CDN Edge Cache                              │
│  - Cache popular prefixes at edge                    │
│  - Reduces latency globally                          │
│  - TTL: 1 hour                                        │
│                                                        │
│  Level 3: Application Cache (Redis)                  │
│  - All prefix → suggestions mappings                 │
│  - TTL: 24 hours                                      │
│                                                        │
│  Cache Warming:                                        │
│  - Pre-compute top 1M prefixes                       │
│  - Load into Redis on startup                        │
│  - Update hourly with trending queries               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 10. Key Takeaways

1. **Trie structure** - efficient prefix lookups
2. **Pre-computed suggestions** - O(1) lookup for common prefixes
3. **Multi-factor ranking** - popularity + trending + personalization
4. **Fuzzy matching** - handle typos with edit distance
5. **Multi-level caching** - browser, CDN, application
6. **Analytics pipeline** - continuously update based on search data

---

Next: [Web Crawler](../07-web-crawler/README.md)
