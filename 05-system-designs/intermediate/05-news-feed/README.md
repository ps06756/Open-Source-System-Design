# Design a News Feed System

Design a personalized news feed system like Facebook's or LinkedIn's feed.

## 1. Requirements

### Functional Requirements
- Users can create posts (text, images, videos)
- Users see a feed of posts from friends/connections
- Feed is ranked by relevance, not just chronological
- Support for likes, comments, shares
- Real-time updates for new posts

### Non-Functional Requirements
- Low latency feed generation (<500ms)
- High availability
- Eventually consistent
- Scale: 1B users, 500M DAU, 10B feed requests/day

### Capacity Estimation

```
Users: 500M DAU
Posts: 50M new posts/day ≈ 600/sec
Feed requests: 10B/day ≈ 115,000/sec
Average user follows: 500 accounts

Storage:
Posts: 50M × 365 = 18B posts/year
Size: 18B × 1KB = 18 TB/year (metadata)
Media: Separate storage (100+ PB)
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Servers     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│                                                 │              │
│                    ┌────────────────────────────┼───────────┐ │
│                    │                            │           │ │
│                    ▼                            ▼           │ │
│             ┌─────────────┐           ┌─────────────────┐   │ │
│             │ Post Service│           │  Feed Service   │   │ │
│             └──────┬──────┘           └────────┬────────┘   │ │
│                    │                           │             │ │
│         ┌──────────┴──────────┐     ┌──────────┴──────────┐ │ │
│         ▼                     ▼     ▼                     ▼ │ │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐ │ │
│  │  Post DB    │      │  Graph DB   │      │ Feed Cache  │ │ │
│  │(PostgreSQL) │      │  (Neo4j)    │      │  (Redis)    │ │ │
│  └─────────────┘      └─────────────┘      └─────────────┘ │ │
│                                                              │ │
│  ┌──────────────────────────────────────────────────────┐   │ │
│  │            Fan-Out Service (Kafka)                    │   │ │
│  └──────────────────────────────────────────────────────┘   │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Feed Generation Approaches

### Push Model (Fan-Out on Write)

```
┌────────────────────────────────────────────────────────┐
│                Fan-Out on Write                        │
│                                                        │
│  When User A posts:                                   │
│  1. Store post in database                            │
│  2. Get all followers of A                           │
│  3. Push post_id to each follower's feed cache       │
│                                                        │
│  User A (1000 followers) posts                       │
│      │                                                │
│      ├──▶ Follower 1 feed cache                      │
│      ├──▶ Follower 2 feed cache                      │
│      ├──▶ ...                                         │
│      └──▶ Follower 1000 feed cache                   │
│                                                        │
│  Pros:                                                │
│  ✓ Fast feed reads (pre-computed)                    │
│  ✓ Simple read path                                  │
│                                                        │
│  Cons:                                                │
│  ✗ Slow writes for popular users                    │
│  ✗ Wasted work if followers don't check feed        │
│  ✗ Hot key problem (celebrities)                    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Pull Model (Fan-Out on Read)

```
┌────────────────────────────────────────────────────────┐
│                Fan-Out on Read                         │
│                                                        │
│  When User B requests feed:                           │
│  1. Get all accounts B follows                        │
│  2. Fetch recent posts from each                     │
│  3. Merge, rank, and return                          │
│                                                        │
│  User B follows 500 accounts                         │
│      │                                                │
│      ├──▶ Query Account 1 posts                      │
│      ├──▶ Query Account 2 posts                      │
│      ├──▶ ...                                         │
│      └──▶ Query Account 500 posts                    │
│           │                                           │
│           └──▶ Merge & Rank                          │
│                                                        │
│  Pros:                                                │
│  ✓ Fast writes                                        │
│  ✓ No wasted computation                             │
│                                                        │
│  Cons:                                                │
│  ✗ Slow reads (many queries)                        │
│  ✗ High read latency                                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Hybrid Model (Recommended)

```
┌────────────────────────────────────────────────────────┐
│                  Hybrid Approach                       │
│                                                        │
│  Regular users (< 10K followers):                     │
│  - Fan-out on write                                   │
│  - Pre-compute feeds                                  │
│                                                        │
│  Celebrity users (> 10K followers):                   │
│  - Fan-out on read                                    │
│  - Merge at read time                                 │
│                                                        │
│  Feed Generation:                                      │
│  1. Get pre-computed feed from cache                 │
│  2. Get celebrity posts at read time                 │
│  3. Merge and rank                                    │
│  4. Apply ML ranking model                           │
│                                                        │
│  def get_feed(user_id):                              │
│      # Pre-computed from regular users               │
│      feed = cache.get(f"feed:{user_id}")             │
│                                                        │
│      # Fetch celebrity posts                          │
│      celebrities = get_followed_celebrities(user_id)  │
│      for celeb in celebrities:                       │
│          posts = get_recent_posts(celeb)             │
│          feed.extend(posts)                          │
│                                                        │
│      # Rank and return                                │
│      return rank(feed)                                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 4. Feed Ranking

### Ranking Model

```
┌────────────────────────────────────────────────────────┐
│                 Feed Ranking                           │
│                                                        │
│  Score = f(affinity, weight, decay)                  │
│                                                        │
│  Affinity: How close is user to post author?         │
│  - Interaction history (likes, comments, messages)   │
│  - Profile visits                                     │
│  - Time spent viewing their posts                    │
│                                                        │
│  Weight: How important is this post type?            │
│  - Post type (photo > text > share)                  │
│  - Engagement (likes, comments, shares)              │
│  - Content quality signals                           │
│                                                        │
│  Decay: How recent is the post?                      │
│  - Exponential decay based on time                   │
│  - decay = e^(-λ × age_hours)                        │
│                                                        │
│  Final Score = Σ(affinity × weight × decay)          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### ML-Based Ranking

```python
class FeedRanker:
    def __init__(self, model):
        self.model = model  # Pre-trained ML model

    def rank(self, user_id, posts):
        user_features = self.get_user_features(user_id)

        scored_posts = []
        for post in posts:
            post_features = self.get_post_features(post)
            author_features = self.get_author_features(post.author_id)

            # Interaction features
            interaction_features = self.get_interaction_features(
                user_id, post.author_id
            )

            # Combine features
            features = {
                **user_features,
                **post_features,
                **author_features,
                **interaction_features,
                'time_since_post': time_since(post.created_at),
            }

            # Predict engagement probability
            score = self.model.predict(features)
            scored_posts.append((post, score))

        # Sort by score
        scored_posts.sort(key=lambda x: -x[1])

        # Diversity injection (avoid showing all posts from same author)
        return self.diversify(scored_posts)
```

## 5. Database Design

### Posts Table

```sql
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    author_id BIGINT NOT NULL,
    content TEXT,
    post_type VARCHAR(20),  -- text, photo, video, share
    media_urls JSONB,
    visibility VARCHAR(20),  -- public, friends, private
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    share_count INT DEFAULT 0,
    created_at TIMESTAMP,
    INDEX idx_author_created (author_id, created_at DESC)
);
```

### Feed Cache (Redis)

```
# Pre-computed feed for each user
Key: feed:{user_id}
Value: Sorted Set of (post_id, timestamp)

# Operations
ZADD feed:123 1640000000 post_789
ZREVRANGE feed:123 0 49  # Get top 50 posts

# With scores for ranking
Key: feed_ranked:{user_id}
Value: Sorted Set of (post_id, score)
```

### Social Graph

```sql
CREATE TABLE follows (
    follower_id BIGINT,
    followee_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);

-- Denormalized follower counts
CREATE TABLE user_stats (
    user_id BIGINT PRIMARY KEY,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    is_celebrity BOOLEAN DEFAULT false  -- > 10K followers
);
```

## 6. Fan-Out Service

```python
class FanOutService:
    def __init__(self, kafka, redis):
        self.kafka = kafka
        self.redis = redis
        self.celebrity_threshold = 10000

    async def fan_out(self, post):
        author = await self.get_user(post.author_id)

        if author.follower_count > self.celebrity_threshold:
            # Celebrity: Don't fan out, will be pulled on read
            return

        # Regular user: Fan out to all followers
        followers = await self.get_followers(post.author_id)

        # Batch updates to Redis
        pipe = self.redis.pipeline()
        for follower_id in followers:
            pipe.zadd(
                f"feed:{follower_id}",
                {str(post.post_id): post.created_at.timestamp()}
            )
            # Trim to keep only recent posts
            pipe.zremrangebyrank(f"feed:{follower_id}", 0, -1001)

        await pipe.execute()
```

## 7. Real-Time Updates

```
┌────────────────────────────────────────────────────────┐
│              Real-Time Feed Updates                    │
│                                                        │
│  Option 1: Long Polling                               │
│  - Client polls every 30 seconds                      │
│  - Server returns new posts since last check          │
│                                                        │
│  Option 2: WebSocket                                   │
│  - Persistent connection                              │
│  - Server pushes new posts immediately               │
│  - More complex but truly real-time                   │
│                                                        │
│  Option 3: Server-Sent Events (SSE)                  │
│  - One-way server → client                           │
│  - Good middle ground                                 │
│                                                        │
│  Implementation (WebSocket):                          │
│                                                        │
│  Client connects → Subscribe to user's feed channel  │
│                                                        │
│  New post created:                                    │
│  1. Fan-out service adds to feed                     │
│  2. Publishes to Redis pub/sub                       │
│  3. WebSocket server receives                        │
│  4. Pushes to connected clients                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 8. Feed Pagination

```python
def get_feed(user_id, cursor=None, limit=20):
    """
    Cursor-based pagination for infinite scroll
    cursor = (score, post_id) - last item from previous page
    """
    feed_key = f"feed:{user_id}"

    if cursor:
        # Get items with score less than cursor
        score, last_id = cursor
        post_ids = redis.zrevrangebyscore(
            feed_key,
            max=f"({score}",  # exclusive
            min="-inf",
            start=0,
            num=limit
        )
    else:
        # First page
        post_ids = redis.zrevrange(feed_key, 0, limit - 1, withscores=True)

    # Hydrate posts
    posts = hydrate_posts(post_ids)

    # Get celebrity posts and merge
    celebrity_posts = get_celebrity_posts(user_id, cursor)
    posts = merge_and_rank(posts, celebrity_posts)

    # Build next cursor
    if posts:
        last_post = posts[-1]
        next_cursor = (last_post.score, last_post.id)
    else:
        next_cursor = None

    return {
        'posts': posts[:limit],
        'next_cursor': next_cursor
    }
```

## 9. Content Deduplication

```
┌────────────────────────────────────────────────────────┐
│             Deduplication Strategies                   │
│                                                        │
│  1. Shared content dedup:                             │
│     - If A shares B's post, and you follow both      │
│     - Show original post with "A shared this"        │
│     - Don't show same content twice                  │
│                                                        │
│  2. Viral content throttling:                         │
│     - If same link shared by many                    │
│     - Group into single card                         │
│     - "50 friends shared this article"               │
│                                                        │
│  3. Same-author grouping:                             │
│     - If author posts 5 times quickly               │
│     - Don't show all 5 consecutively                │
│     - Space them out in feed                         │
│                                                        │
│  Implementation:                                       │
│  - Track content hashes                              │
│  - Bloom filter for quick lookup                     │
│  - Collapse duplicates in ranking phase             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 10. Key Takeaways

1. **Hybrid fan-out** - push for regular users, pull for celebrities
2. **ML-based ranking** - personalize feed based on user behavior
3. **Pre-computed feeds** - fast reads from cache
4. **Cursor pagination** - efficient infinite scroll
5. **Real-time updates** - WebSocket or SSE for live feed
6. **Content deduplication** - avoid showing same content twice

---

Next: [Typeahead](../06-typeahead/README.md)
