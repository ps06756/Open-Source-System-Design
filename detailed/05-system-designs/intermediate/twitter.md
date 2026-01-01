# Design Twitter

## 1. Requirements

### Functional Requirements
- Post tweets (280 characters, images, videos)
- Follow/unfollow users
- View home timeline (tweets from followed users)
- View user timeline (user's own tweets)
- Like, retweet, reply to tweets
- Search tweets and users

### Non-Functional Requirements
- High availability
- Eventual consistency acceptable
- Low latency for timeline reads (< 200ms)
- Handle viral tweets (celebrity problem)

### Scale Estimates
- 500M daily active users
- 200M tweets per day
- 100B timeline reads per day
- Average user follows 200 users

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Tweets: 200M/day = 2,300/second                           │
│  Timeline reads: 100B/day = 1.15M/second                   │
│  Read:Write ratio = ~500:1 (extremely read-heavy)          │
│  Average tweet size: 500 bytes (with metadata)             │
│  Storage: 200M × 500B × 365 = 36 TB/year (tweets only)    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐                                             │
│  │   Client   │                                             │
│  └─────┬──────┘                                             │
│        │                                                     │
│        ▼                                                     │
│  ┌────────────┐     ┌─────────────────────────────────────┐ │
│  │    CDN     │────▶│          Load Balancer              │ │
│  └────────────┘     └──────────────┬──────────────────────┘ │
│                                    │                        │
│         ┌──────────────────────────┼──────────────────┐     │
│         ▼                          ▼                  ▼     │
│  ┌─────────────┐        ┌─────────────┐       ┌──────────┐ │
│  │   Tweet     │        │  Timeline   │       │  User    │ │
│  │  Service    │        │  Service    │       │ Service  │ │
│  └──────┬──────┘        └──────┬──────┘       └────┬─────┘ │
│         │                      │                   │        │
│         ▼                      ▼                   ▼        │
│  ┌─────────────┐        ┌─────────────┐       ┌──────────┐ │
│  │Tweet Store  │        │ Timeline    │       │User Store│ │
│  │(Cassandra)  │        │ Cache       │       │(MySQL)   │ │
│  └─────────────┘        │ (Redis)     │       └──────────┘ │
│         │               └─────────────┘              │      │
│         │                                            │      │
│         └───────────────────┬────────────────────────┘      │
│                             ▼                               │
│                    ┌─────────────────┐                      │
│                    │   Fan-out       │                      │
│                    │   Service       │                      │
│                    └─────────────────┘                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. The Fan-out Problem

```
┌─────────────────────────────────────────────────────────────┐
│                   The Fan-out Problem                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  When Taylor Swift (100M followers) tweets:                │
│                                                              │
│  Approach 1: Fan-out on Write (Push Model)                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Tweet → Push to 100M follower timelines               │ │
│  │                                                         │ │
│  │  ✓ Fast timeline reads (pre-computed)                  │ │
│  │  ✗ Slow writes for celebrities                        │ │
│  │  ✗ 100M write operations per tweet!                   │ │
│  │  ✗ Wasted if followers don't check timeline           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Approach 2: Fan-out on Read (Pull Model)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Timeline read → Fetch from all followed users         │ │
│  │                                                         │ │
│  │  ✓ Fast writes                                         │ │
│  │  ✗ Slow timeline reads                                │ │
│  │  ✗ If following 500 users, need 500 queries!         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Approach 3: Hybrid (What Twitter Uses)                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Regular users (< 10K followers): Fan-out on write  │ │
│  │  • Celebrities (> 10K followers): Fan-out on read    │ │
│  │                                                         │ │
│  │  Timeline = Pre-computed feed + Celebrity tweets      │ │
│  │                                                         │ │
│  │  ✓ Balances read and write performance               │ │
│  │  ✓ Celebrity tweets fetched at read time             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Hybrid Fan-out Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               Hybrid Fan-out Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Post Tweet:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌──────────┐    Is celebrity?                         │ │
│  │  │  Tweet   │──────────────────────────────────┐       │ │
│  │  └────┬─────┘                                  │       │ │
│  │       │ No                                     │ Yes   │ │
│  │       ▼                                        ▼       │ │
│  │  ┌─────────────┐                      ┌─────────────┐ │ │
│  │  │  Fan-out    │                      │ Store only  │ │ │
│  │  │  to follower│                      │ (no fanout) │ │ │
│  │  │  timelines  │                      └─────────────┘ │ │
│  │  └─────────────┘                                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Read Timeline:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌─────────────────┐     ┌─────────────────────────┐  │ │
│  │  │ Pre-computed    │     │ Followed celebrities'   │  │ │
│  │  │ timeline cache  │  +  │ recent tweets           │  │ │
│  │  └─────────────────┘     └─────────────────────────┘  │ │
│  │           │                        │                   │ │
│  │           └────────────┬───────────┘                   │ │
│  │                        ▼                               │ │
│  │                ┌───────────────┐                       │ │
│  │                │ Merge & Sort  │                       │ │
│  │                │ by timestamp  │                       │ │
│  │                └───────────────┘                       │ │
│  │                        │                               │ │
│  │                        ▼                               │ │
│  │                   Return timeline                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Users Table (MySQL - need ACID for user data)             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ user_id (PK)    │ BIGINT                                │ │
│  │ username        │ VARCHAR(50) UNIQUE                    │ │
│  │ email           │ VARCHAR(255)                          │ │
│  │ display_name    │ VARCHAR(100)                          │ │
│  │ bio             │ TEXT                                  │ │
│  │ followers_count │ INT                                   │ │
│  │ following_count │ INT                                   │ │
│  │ is_celebrity    │ BOOLEAN (followers > 10K)            │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Tweets Table (Cassandra - write-heavy, time-series)       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ tweet_id (PK)   │ BIGINT (Snowflake ID)                │ │
│  │ user_id         │ BIGINT                                │ │
│  │ content         │ TEXT (280 chars)                      │ │
│  │ media_urls      │ LIST<TEXT>                            │ │
│  │ reply_to        │ BIGINT (nullable)                     │ │
│  │ retweet_of      │ BIGINT (nullable)                     │ │
│  │ likes_count     │ COUNTER                               │ │
│  │ retweets_count  │ COUNTER                               │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  User Timeline (Redis - list of tweet IDs)                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Key: timeline:{user_id}                                 │ │
│  │ Value: List of tweet_ids (most recent first)           │ │
│  │ Size: Last 800 tweets                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Follows Table (Graph relationship)                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ follower_id     │ BIGINT                                │ │
│  │ followee_id     │ BIGINT                                │ │
│  │ created_at      │ TIMESTAMP                             │ │
│  │ PRIMARY KEY (follower_id, followee_id)                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### ID Generation (Snowflake)

```
┌─────────────────────────────────────────────────────────────┐
│               Snowflake ID Generation                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  64-bit ID structure:                                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1 bit │   41 bits    │  10 bits  │     12 bits         ││
│  │ sign  │  timestamp   │ machine ID│   sequence number   ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  • 41 bits timestamp: ~69 years of milliseconds            │
│  • 10 bits machine: 1024 machines                          │
│  • 12 bits sequence: 4096 IDs per millisecond per machine │
│                                                              │
│  Properties:                                                 │
│  ✓ Time-ordered (can sort by ID)                           │
│  ✓ Unique across distributed systems                       │
│  ✓ No coordination needed                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Timeline Generation

```
┌─────────────────────────────────────────────────────────────┐
│                 Timeline Generation                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User requests timeline:                                    │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Fetch pre-computed timeline from Redis             │ │
│  │     timeline:{user_id} → [tweet_id1, tweet_id2, ...]  │ │
│  │                                                         │ │
│  │  2. Get list of followed celebrities                   │ │
│  │     followed_celebrities:{user_id} → [celeb1, celeb2] │ │
│  │                                                         │ │
│  │  3. Fetch recent tweets from each celebrity           │ │
│  │     user_tweets:{celeb_id} → [recent tweets]          │ │
│  │                                                         │ │
│  │  4. Merge and sort by timestamp                        │ │
│  │     merged = sort(precomputed + celebrity_tweets)      │ │
│  │                                                         │ │
│  │  5. Hydrate tweet objects                              │ │
│  │     MGET tweet:{id1} tweet:{id2} ...                  │ │
│  │                                                         │ │
│  │  6. Return to client                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Timeline Cache Structure:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Key: timeline:{user_id}                                │ │
│  │  Type: Sorted Set (ZSET)                               │ │
│  │  Score: Tweet timestamp (for ordering)                 │ │
│  │  Member: Tweet ID                                       │ │
│  │  Size: 800 most recent tweets                          │ │
│  │                                                         │ │
│  │  Operations:                                            │ │
│  │  - ZADD timeline:123 {timestamp} {tweet_id}           │ │
│  │  - ZREVRANGE timeline:123 0 49  # Get latest 50       │ │
│  │  - ZREMRANGEBYRANK timeline:123 0 -801  # Trim to 800 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Search

```
┌─────────────────────────────────────────────────────────────┐
│                        Search                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Search Architecture:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Tweet Posted                                           │ │
│  │       │                                                 │ │
│  │       ▼                                                 │ │
│  │  ┌─────────────┐                                       │ │
│  │  │ Tweet Store │──────────────────────────────────┐    │ │
│  │  │ (Primary)   │                                  │    │ │
│  │  └─────────────┘                                  │    │ │
│  │       │                                           │    │ │
│  │       │ Async                                     │    │ │
│  │       ▼                                           ▼    │ │
│  │  ┌─────────────┐                          ┌───────────┐│ │
│  │  │ Search Index│                          │ Analytics ││ │
│  │  │(Elasticsearch)                         │ (Spark)   ││ │
│  │  └─────────────┘                          └───────────┘│ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Elasticsearch Index:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  {                                                      │ │
│  │    "mappings": {                                        │ │
│  │      "properties": {                                    │ │
│  │        "tweet_id": { "type": "keyword" },              │ │
│  │        "user_id": { "type": "keyword" },               │ │
│  │        "content": { "type": "text", "analyzer": "std" }│ │
│  │        "hashtags": { "type": "keyword" },              │ │
│  │        "created_at": { "type": "date" }               │ │
│  │      }                                                  │ │
│  │    }                                                    │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Search ranking factors:                                    │
│  • Text relevance                                           │
│  • Recency                                                   │
│  • Engagement (likes, retweets)                            │
│  • User popularity                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Timeline Cache** | Redis | Fast sorted sets, TTL |
| **Tweet Storage** | Cassandra | Write-heavy, time-series |
| **User Storage** | MySQL | ACID, relationships |
| **Search** | Elasticsearch | Full-text search |
| **Fan-out** | Hybrid push/pull | Balance write/read |
| **ID Generation** | Snowflake | Time-ordered, distributed |

---

[Back to System Designs](../README.md)
