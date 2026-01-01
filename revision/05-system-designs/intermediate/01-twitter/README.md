# Design Twitter / X

Design a social media platform like Twitter where users can post tweets and follow others.

## 1. Requirements

### Functional Requirements
- Post tweets (280 characters)
- Follow/unfollow users
- View home timeline (tweets from followed users)
- View user timeline (tweets from specific user)
- Like and retweet
- Search tweets

### Non-Functional Requirements
- High availability
- Low latency for timeline (<200ms)
- Eventual consistency is acceptable
- Scale: 500M DAU, 500M tweets/day

### Capacity Estimation

```
Users: 500M DAU
Tweets: 500M/day ≈ 6,000 tweets/sec
Timeline reads: 500M × 10 views/day = 5B/day ≈ 60,000/sec

Storage (5 years):
Tweets: 500M × 365 × 5 = 900B tweets
Size: 900B × 500 bytes = 450 TB

Media: Assume 10% tweets have media
= 90B × 1 MB = 90 PB
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Servers     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│         ┌───────────────────────────────────────┼───────────┐ │
│         │                                       │           │ │
│         ▼                                       ▼           │ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │ │
│  │   Cache     │  │   Tweet     │  │    User/Graph       │ │ │
│  │  (Redis)    │  │  Database   │  │     Database        │ │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │ │
│                                                              │ │
│  ┌──────────────────────────────────────────────────────┐   │ │
│  │              Timeline Service                         │   │ │
│  │  (Fan-out tweets to follower timelines)              │   │ │
│  └──────────────────────────────────────────────────────┘   │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. The Fan-Out Problem

### Fan-Out on Write (Push)

```
User A posts tweet → Push to all followers' timelines

┌────────────────────────────────────────────────────────┐
│                                                        │
│  User A (1M followers) posts tweet                    │
│                                                        │
│  ┌────────────┐                                       │
│  │  Tweet     │──────┬──────────────────────────────▶ │
│  │ "Hello!"   │      │                                │
│  └────────────┘      │    Write to 1M timelines      │
│                      │                                │
│              ┌───────┴────────┐                      │
│              ▼                ▼                      │
│        ┌──────────┐    ┌──────────┐                 │
│        │Follower 1│    │Follower 2│  ... 1M times   │
│        │ Timeline │    │ Timeline │                 │
│        └──────────┘    └──────────┘                 │
│                                                        │
│  Pros: Fast timeline reads                            │
│  Cons: Slow writes for popular users                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Fan-Out on Read (Pull)

```
User views timeline → Fetch tweets from followed users

┌────────────────────────────────────────────────────────┐
│                                                        │
│  User B (follows 500 users) requests timeline         │
│                                                        │
│  ┌────────────┐                                       │
│  │  Request   │──────┬──────────────────────────────▶ │
│  │  Timeline  │      │                                │
│  └────────────┘      │    Query 500 users' tweets    │
│                      │    Merge and sort             │
│              ┌───────┴────────┐                      │
│              ▼                ▼                      │
│        ┌──────────┐    ┌──────────┐                 │
│        │ User 1's │    │ User 2's │  ... 500 queries│
│        │  Tweets  │    │  Tweets  │                 │
│        └──────────┘    └──────────┘                 │
│                                                        │
│  Pros: Fast writes                                    │
│  Cons: Slow timeline reads                           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Hybrid Approach (Twitter's Solution)

```
┌────────────────────────────────────────────────────────┐
│                  Hybrid Fan-Out                        │
│                                                        │
│  Regular users (< 10K followers): Fan-out on write   │
│  - Pre-compute timelines                              │
│  - Fast reads                                          │
│                                                        │
│  Celebrity users (> 10K followers): Fan-out on read  │
│  - Don't pre-compute                                  │
│  - Merge at read time                                 │
│                                                        │
│  Timeline = pre-computed + celebrity tweets           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 4. Database Design

### Tweet Table

```sql
CREATE TABLE tweets (
    tweet_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content VARCHAR(280),
    media_urls TEXT[],
    created_at TIMESTAMP,
    like_count INT DEFAULT 0,
    retweet_count INT DEFAULT 0
);

-- Sharded by tweet_id
```

### Timeline Cache (Redis)

```
Key: timeline:{user_id}
Value: Sorted Set of tweet_ids by timestamp

ZADD timeline:123 1640000000 tweet_789
ZADD timeline:123 1640000100 tweet_790

ZREVRANGE timeline:123 0 20  -- Get latest 20 tweets
```

### User Graph (Neo4j or Cassandra)

```
CREATE TABLE followers (
    user_id BIGINT,
    follower_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, follower_id)
);

CREATE TABLE following (
    user_id BIGINT,
    following_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, following_id)
);
```

## 5. Post Tweet Flow

```
1. Client posts tweet
2. API server validates and stores tweet
3. Send to fan-out service via message queue
4. Fan-out service:
   a. Get followers list
   b. For each follower (< 10K followers):
      - Add tweet_id to their timeline cache
5. Return success to client
```

```python
def post_tweet(user_id, content):
    # Store tweet
    tweet_id = generate_snowflake_id()
    db.insert('tweets', {
        'tweet_id': tweet_id,
        'user_id': user_id,
        'content': content,
        'created_at': now()
    })

    # Async fan-out
    queue.publish('fanout', {
        'tweet_id': tweet_id,
        'user_id': user_id
    })

    return tweet_id
```

## 6. Get Timeline Flow

```python
def get_timeline(user_id, cursor=None, limit=20):
    # Get pre-computed timeline
    tweet_ids = cache.zrevrange(
        f'timeline:{user_id}',
        cursor or 0,
        limit
    )

    # Get celebrity tweets (fan-out on read)
    celebrities = get_followed_celebrities(user_id)
    celebrity_tweets = []
    for celeb_id in celebrities:
        tweets = get_recent_tweets(celeb_id, limit=5)
        celebrity_tweets.extend(tweets)

    # Merge and sort
    all_tweets = tweet_ids + [t.id for t in celebrity_tweets]
    all_tweets.sort(key=lambda t: t.timestamp, reverse=True)

    # Hydrate tweet data
    return hydrate_tweets(all_tweets[:limit])
```

## 7. ID Generation (Snowflake)

```
┌────────────────────────────────────────────────────────┐
│              Snowflake ID (64 bits)                   │
│                                                        │
│  ┌─────┬─────────────────┬────────────┬─────────────┐ │
│  │  1  │   41 bits       │  10 bits   │  12 bits    │ │
│  │sign │   timestamp     │ machine ID │  sequence   │ │
│  └─────┴─────────────────┴────────────┴─────────────┘ │
│                                                        │
│  - Time-ordered (sortable)                            │
│  - Unique across machines                             │
│  - 4096 IDs per millisecond per machine              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 8. Search

```
┌────────────────────────────────────────────────────────┐
│                    Search                              │
│                                                        │
│  Elasticsearch cluster                                 │
│                                                        │
│  Index: tweets                                         │
│  - content (full-text)                                │
│  - hashtags                                            │
│  - user mentions                                       │
│  - timestamp                                           │
│                                                        │
│  Real-time indexing via Kafka                         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Hybrid fan-out** - push for regular users, pull for celebrities
2. **Timeline in Redis** - sorted set for fast access
3. **Snowflake IDs** - time-ordered, distributed
4. **Async processing** - fan-out via message queue
5. **Separate read/write paths** - optimize each independently

---

Next: [Instagram](../02-instagram/README.md)
