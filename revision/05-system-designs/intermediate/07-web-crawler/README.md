# Design a Web Crawler

Design a web crawler that indexes the internet for a search engine.

## 1. Requirements

### Functional Requirements
- Crawl web pages starting from seed URLs
- Extract and follow links
- Store page content for indexing
- Handle robots.txt and politeness
- Detect and handle duplicate pages
- Support incremental/continuous crawling

### Non-Functional Requirements
- Scalability: Crawl billions of pages
- Politeness: Don't overwhelm websites
- Robustness: Handle failures gracefully
- Extensibility: Support different content types
- Freshness: Re-crawl pages periodically

### Capacity Estimation

```
Target: Crawl 1B pages/month
Pages/day: 33M
Pages/sec: ~400

Average page size: 500 KB
Storage/month: 1B × 500 KB = 500 TB
Storage/year: 6 PB

Bandwidth: 400 × 500 KB = 200 MB/s
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌──────────────┐     ┌──────────────┐     ┌───────────────┐  │
│  │  URL Frontier│────▶│   Fetcher    │────▶│    Parser     │  │
│  │   (Queue)    │     │   Workers    │     │   Workers     │  │
│  └──────────────┘     └──────────────┘     └───────┬───────┘  │
│         ▲                                          │          │
│         │                                          │          │
│         │              ┌───────────────────────────┘          │
│         │              │                                       │
│         │              ▼                                       │
│         │       ┌─────────────┐     ┌────────────────────┐    │
│         │       │   URL       │     │   Content Store    │    │
│         └───────│  Extractor  │     │   (S3 + Index)     │    │
│                 └─────────────┘     └────────────────────┘    │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                   Support Services                        │ │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │ │
│  │  │ DNS Cache  │  │  Robots.txt│  │  Duplicate         │  │ │
│  │  │            │  │   Cache    │  │  Detector          │  │ │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. URL Frontier

```
┌────────────────────────────────────────────────────────────────┐
│                       URL Frontier                             │
│                                                                │
│  Priority-based queue with politeness constraints             │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Front Queues (Priority)                                  │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐                        │ │
│  │  │High Pri│ │Med Pri │ │Low Pri │                        │ │
│  │  │  .com  │ │  .org  │ │  .edu  │                        │ │
│  │  └───┬────┘ └───┬────┘ └───┬────┘                        │ │
│  │      │          │          │                              │ │
│  │      └──────────┴──────────┘                              │ │
│  │                  │                                        │ │
│  │                  ▼                                        │ │
│  │  ┌────────────────────────────────────────────────────┐  │ │
│  │  │              Router (by domain)                     │  │ │
│  │  └────────────────────────────────────────────────────┘  │ │
│  │                  │                                        │ │
│  │                  ▼                                        │ │
│  │  Back Queues (Per-domain for politeness)                 │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │ │
│  │  │google  │ │amazon  │ │wiki    │ │  ...   │           │ │
│  │  │.com    │ │.com    │ │pedia   │ │        │           │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘           │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  Politeness: One request per domain every N seconds          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### URL Prioritization

```python
class URLFrontier:
    def __init__(self):
        self.priority_queues = {
            'high': PriorityQueue(),
            'medium': PriorityQueue(),
            'low': PriorityQueue()
        }
        self.domain_queues = {}
        self.domain_last_access = {}
        self.politeness_delay = 1.0  # seconds

    def add_url(self, url, priority='medium'):
        """Add URL to frontier"""
        # Check if already crawled or in queue
        if self.is_duplicate(url):
            return

        # Add to priority queue
        score = self.calculate_priority(url)
        self.priority_queues[priority].put((score, url))

    def get_next_url(self):
        """Get next URL respecting politeness"""
        # Try high priority first
        for priority in ['high', 'medium', 'low']:
            queue = self.priority_queues[priority]

            while not queue.empty():
                score, url = queue.get()
                domain = get_domain(url)

                # Check politeness
                last_access = self.domain_last_access.get(domain, 0)
                if time.time() - last_access >= self.politeness_delay:
                    self.domain_last_access[domain] = time.time()
                    return url
                else:
                    # Put back and try another
                    queue.put((score, url))
                    continue

        return None

    def calculate_priority(self, url):
        """Higher score = higher priority"""
        score = 0

        # PageRank or similar importance metric
        score += self.get_page_importance(url) * 100

        # Freshness (when was it last crawled?)
        last_crawl = self.get_last_crawl_time(url)
        if last_crawl:
            age_days = (time.time() - last_crawl) / 86400
            score += min(age_days, 30)  # Cap at 30 days
        else:
            score += 50  # New URLs get boost

        return score
```

## 4. Fetcher

```python
class Fetcher:
    def __init__(self):
        self.session = aiohttp.ClientSession()
        self.robots_cache = RobotsCache()
        self.dns_cache = DNSCache()

    async def fetch(self, url):
        """Fetch a URL with proper handling"""
        domain = get_domain(url)

        # Check robots.txt
        robots = await self.robots_cache.get(domain)
        if not robots.can_fetch('*', url):
            return None

        # Resolve DNS (with caching)
        ip = await self.dns_cache.resolve(domain)

        try:
            async with self.session.get(
                url,
                timeout=30,
                headers={'User-Agent': 'SearchBot/1.0'},
                allow_redirects=True,
                max_redirects=5
            ) as response:

                if response.status != 200:
                    return None

                # Check content type
                content_type = response.headers.get('Content-Type', '')
                if 'text/html' not in content_type:
                    return None

                # Limit size (avoid huge files)
                content = await response.read()
                if len(content) > 10 * 1024 * 1024:  # 10 MB limit
                    return None

                return {
                    'url': str(response.url),
                    'content': content,
                    'headers': dict(response.headers),
                    'status': response.status
                }

        except Exception as e:
            logger.error(f"Failed to fetch {url}: {e}")
            return None
```

### Robots.txt Handling

```python
class RobotsCache:
    def __init__(self, redis):
        self.redis = redis
        self.parser = urllib.robotparser.RobotFileParser()

    async def get(self, domain):
        """Get and cache robots.txt for domain"""
        cache_key = f"robots:{domain}"

        # Check cache
        cached = await self.redis.get(cache_key)
        if cached:
            return self.parse_robots(cached)

        # Fetch robots.txt
        robots_url = f"https://{domain}/robots.txt"
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(robots_url, timeout=10) as resp:
                    if resp.status == 200:
                        content = await resp.text()
                    else:
                        content = ""  # Allow all if no robots.txt
        except:
            content = ""

        # Cache for 24 hours
        await self.redis.set(cache_key, content, ex=86400)

        return self.parse_robots(content)
```

## 5. Parser

```python
class Parser:
    def parse(self, page_data):
        """Parse HTML and extract content"""
        soup = BeautifulSoup(page_data['content'], 'lxml')

        # Extract text content
        text = self.extract_text(soup)

        # Extract links
        links = self.extract_links(soup, page_data['url'])

        # Extract metadata
        metadata = self.extract_metadata(soup)

        return {
            'url': page_data['url'],
            'title': metadata.get('title', ''),
            'description': metadata.get('description', ''),
            'text': text,
            'links': links,
            'crawled_at': time.time()
        }

    def extract_links(self, soup, base_url):
        """Extract and normalize all links"""
        links = []

        for a in soup.find_all('a', href=True):
            href = a['href']

            # Normalize URL
            absolute_url = urljoin(base_url, href)

            # Filter out unwanted URLs
            if self.is_valid_url(absolute_url):
                links.append(absolute_url)

        return list(set(links))  # Dedupe

    def is_valid_url(self, url):
        """Check if URL should be crawled"""
        parsed = urlparse(url)

        # Only HTTP/HTTPS
        if parsed.scheme not in ['http', 'https']:
            return False

        # Skip common non-HTML extensions
        skip_extensions = ['.pdf', '.jpg', '.png', '.gif', '.zip', '.exe']
        if any(parsed.path.lower().endswith(ext) for ext in skip_extensions):
            return False

        return True
```

## 6. Duplicate Detection

```
┌────────────────────────────────────────────────────────────────┐
│                   Duplicate Detection                          │
│                                                                │
│  Two types of duplicates:                                     │
│                                                                │
│  1. URL Duplicates                                            │
│     - Same URL seen before                                    │
│     - Solution: URL hash set (Bloom filter)                  │
│                                                                │
│  2. Content Duplicates                                        │
│     - Different URLs, same content                           │
│     - Solution: Content fingerprinting (SimHash)             │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   SimHash Algorithm                      │  │
│  │                                                          │  │
│  │  1. Extract features (words, n-grams)                   │  │
│  │  2. Hash each feature                                    │  │
│  │  3. Weight by frequency                                  │  │
│  │  4. Sum weighted hashes                                  │  │
│  │  5. Generate fingerprint                                 │  │
│  │                                                          │  │
│  │  Similar documents have similar fingerprints            │  │
│  │  Hamming distance < 3 = likely duplicate                │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### SimHash Implementation

```python
class SimHash:
    def __init__(self, bits=64):
        self.bits = bits

    def compute(self, text):
        """Compute SimHash fingerprint"""
        # Extract features (words)
        words = text.lower().split()

        # Initialize vector
        v = [0] * self.bits

        for word in words:
            # Hash the word
            word_hash = self.hash_word(word)

            # Update vector
            for i in range(self.bits):
                if word_hash & (1 << i):
                    v[i] += 1
                else:
                    v[i] -= 1

        # Generate fingerprint
        fingerprint = 0
        for i in range(self.bits):
            if v[i] > 0:
                fingerprint |= (1 << i)

        return fingerprint

    def hamming_distance(self, hash1, hash2):
        """Count differing bits"""
        xor = hash1 ^ hash2
        return bin(xor).count('1')

    def is_duplicate(self, hash1, hash2, threshold=3):
        """Check if two pages are duplicates"""
        return self.hamming_distance(hash1, hash2) <= threshold

class DuplicateDetector:
    def __init__(self):
        self.url_bloom = BloomFilter(capacity=10_000_000_000)
        self.simhash = SimHash()
        self.fingerprints = {}  # Or use LSH for scale

    def is_url_seen(self, url):
        """Check if URL was crawled before"""
        url_hash = hashlib.md5(url.encode()).hexdigest()

        if url_hash in self.url_bloom:
            return True

        self.url_bloom.add(url_hash)
        return False

    def is_content_duplicate(self, content):
        """Check if content is duplicate"""
        fingerprint = self.simhash.compute(content)

        # Check against existing fingerprints
        for existing in self.fingerprints.values():
            if self.simhash.is_duplicate(fingerprint, existing):
                return True

        return False
```

## 7. Storage

```
┌────────────────────────────────────────────────────────────────┐
│                      Storage Strategy                          │
│                                                                │
│  Raw Content:                                                  │
│  - Store in S3 or similar object storage                      │
│  - Compress (gzip)                                            │
│  - Key: hash(url)                                             │
│                                                                │
│  Metadata (PostgreSQL):                                        │
│  CREATE TABLE pages (                                          │
│      url_hash VARCHAR(64) PRIMARY KEY,                        │
│      url TEXT,                                                 │
│      title TEXT,                                               │
│      content_hash VARCHAR(64),                                │
│      crawled_at TIMESTAMP,                                    │
│      last_modified TIMESTAMP,                                 │
│      status_code INT,                                          │
│      content_length INT                                        │
│  );                                                            │
│                                                                │
│  Links (for PageRank):                                         │
│  CREATE TABLE links (                                          │
│      source_url_hash VARCHAR(64),                             │
│      target_url_hash VARCHAR(64),                             │
│      PRIMARY KEY (source_url_hash, target_url_hash)           │
│  );                                                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. Distributed Architecture

```
┌────────────────────────────────────────────────────────────────┐
│              Distributed Crawler Architecture                  │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                    Coordinator                          │   │
│  │  - Manages URL frontier                                │   │
│  │  - Assigns URLs to workers                             │   │
│  │  - Tracks progress                                      │   │
│  └─────────────────────────┬──────────────────────────────┘   │
│                            │                                   │
│         ┌──────────────────┼──────────────────┐               │
│         │                  │                  │               │
│         ▼                  ▼                  ▼               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │  Worker 1   │    │  Worker 2   │    │  Worker N   │       │
│  │ (US-East)   │    │ (EU-West)   │    │ (Asia)      │       │
│  │             │    │             │    │             │       │
│  │ Fetch/Parse │    │ Fetch/Parse │    │ Fetch/Parse │       │
│  └─────────────┘    └─────────────┘    └─────────────┘       │
│                                                                │
│  Geographic distribution:                                     │
│  - Workers near target websites = lower latency              │
│  - Assign .cn domains to Asia workers                        │
│  - Assign .eu domains to EU workers                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### URL Partitioning

```python
def assign_url_to_worker(url):
    """Partition URLs across workers by domain hash"""
    domain = get_domain(url)

    # Consistent hashing for distribution
    hash_value = hash(domain) % num_workers

    # Or geographic-based assignment
    if domain.endswith('.cn') or domain.endswith('.jp'):
        return 'asia-worker-pool'
    elif domain.endswith('.eu') or domain.endswith('.de'):
        return 'eu-worker-pool'
    else:
        return 'us-worker-pool'
```

## 9. Recrawling Strategy

```
┌────────────────────────────────────────────────────────────────┐
│                   Recrawl Strategy                             │
│                                                                │
│  Different pages need different refresh rates:                │
│                                                                │
│  News sites: Every few hours                                  │
│  Blogs: Daily                                                  │
│  Wikipedia: Weekly                                            │
│  Static pages: Monthly                                        │
│                                                                │
│  Adaptive strategy:                                            │
│  - Track how often page changes                               │
│  - Increase/decrease crawl frequency accordingly             │
│                                                                │
│  def calculate_recrawl_interval(page):                        │
│      # Based on historical change rate                        │
│      change_rate = page.changes / page.crawl_count           │
│                                                                │
│      if change_rate > 0.8:  # Changes 80% of time           │
│          return timedelta(hours=6)                            │
│      elif change_rate > 0.5:                                  │
│          return timedelta(days=1)                             │
│      elif change_rate > 0.1:                                  │
│          return timedelta(weeks=1)                            │
│      else:                                                     │
│          return timedelta(weeks=4)                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 10. Key Takeaways

1. **URL Frontier** - priority queues with per-domain politeness
2. **Robots.txt compliance** - respect website crawl rules
3. **Duplicate detection** - Bloom filter for URLs, SimHash for content
4. **Distributed workers** - geographic distribution for latency
5. **Adaptive recrawling** - frequency based on change rate
6. **Fault tolerance** - checkpoint progress, retry failed URLs

---

[Back to Intermediate](../README.md) | Next: [Advanced Designs](../../advanced/README.md)
