# Design Instagram

Design a photo and video sharing social network like Instagram.

## 1. Requirements

### Functional Requirements
- Upload photos and videos
- Follow/unfollow users
- View news feed (posts from followed users)
- Like and comment on posts
- Search users and hashtags
- Stories (24-hour ephemeral content)

### Non-Functional Requirements
- High availability
- Low latency feed loading (<500ms)
- Eventual consistency acceptable
- Scale: 1B+ users, 500M DAU, 100M uploads/day

### Capacity Estimation

```
Users: 500M DAU
Uploads: 100M photos/day ≈ 1,200/sec
Feed reads: 500M × 20 views/day = 10B/day ≈ 115,000/sec

Storage (5 years):
Photos: 100M × 365 × 5 = 180B photos
Average photo: 2 MB (multiple resolutions)
Total: 180B × 2 MB = 360 PB

Bandwidth:
Upload: 1,200 × 2 MB = 2.4 GB/s
Download: 115,000 × 500 KB = 57 GB/s
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
│  │    CDN      │  │   Object    │  │    Metadata DB      │ │ │
│  │  (Images)   │  │   Storage   │  │    (PostgreSQL)     │ │ │
│  └─────────────┘  │   (S3)      │  └─────────────────────┘ │ │
│                   └─────────────┘                           │ │
│  ┌──────────────────────────────────────────────────────┐   │ │
│  │              Feed Generation Service                  │   │ │
│  │  (Fanout + Ranking)                                  │   │ │
│  └──────────────────────────────────────────────────────┘   │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Photo Upload Flow

```
┌────────────────────────────────────────────────────────┐
│                    Upload Flow                         │
│                                                        │
│  1. Client uploads photo                              │
│  2. Generate unique ID (Snowflake)                   │
│  3. Store original in S3                              │
│  4. Queue image processing job                        │
│  5. Return success to client                          │
│                                                        │
│  Background:                                           │
│  6. Generate thumbnails (150px, 350px, 640px, 1080px)│
│  7. Store resized images in S3                       │
│  8. Update CDN                                         │
│  9. Fan-out to followers' feeds                       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Image Processing Pipeline

```python
def upload_photo(user_id, image_data, caption, tags):
    photo_id = generate_snowflake_id()

    # Store original
    original_key = f"photos/{photo_id}/original"
    s3.put_object(Bucket="instagram-photos", Key=original_key, Body=image_data)

    # Store metadata
    db.insert('posts', {
        'post_id': photo_id,
        'user_id': user_id,
        'caption': caption,
        'tags': tags,
        'status': 'processing',
        'created_at': now()
    })

    # Queue processing
    queue.publish('image-processing', {
        'photo_id': photo_id,
        'original_key': original_key
    })

    return photo_id

def process_image(photo_id, original_key):
    original = s3.get_object(original_key)

    sizes = [
        ('thumb', 150, 150),
        ('small', 350, 350),
        ('medium', 640, 640),
        ('large', 1080, 1080)
    ]

    for name, width, height in sizes:
        resized = resize_image(original, width, height)
        key = f"photos/{photo_id}/{name}"
        s3.put_object(Bucket="instagram-photos", Key=key, Body=resized)

    # Update status
    db.update('posts', {'post_id': photo_id}, {'status': 'active'})

    # Fan-out to feeds
    fan_out_to_followers(photo_id)
```

## 4. Feed Generation

### Hybrid Approach (Like Twitter)

```
┌────────────────────────────────────────────────────────┐
│                Feed Generation                         │
│                                                        │
│  Regular Users (< 10K followers):                     │
│  - Fan-out on write                                   │
│  - Pre-compute feeds                                  │
│  - Fast reads                                          │
│                                                        │
│  Celebrity Users (> 10K followers):                   │
│  - Fan-out on read                                    │
│  - Merge at request time                              │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │  User Feed = Pre-computed + Celebrity Posts    │   │
│  │            + Ranked by ML Model                │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Feed Ranking

```python
def get_feed(user_id, cursor=None, limit=20):
    # Get pre-computed feed
    post_ids = cache.zrevrange(f'feed:{user_id}', cursor or 0, limit * 2)

    # Get celebrity posts
    celebrities = get_followed_celebrities(user_id)
    celebrity_posts = []
    for celeb_id in celebrities:
        posts = get_recent_posts(celeb_id, since=cursor_timestamp)
        celebrity_posts.extend(posts)

    # Combine and rank
    all_posts = hydrate_posts(post_ids + [p.id for p in celebrity_posts])
    ranked_posts = rank_posts(user_id, all_posts)

    return ranked_posts[:limit]

def rank_posts(user_id, posts):
    """
    Ranking factors:
    - Recency
    - Engagement (likes, comments)
    - User relationship strength
    - Content type preference
    - Past interactions
    """
    user_features = get_user_features(user_id)

    scored_posts = []
    for post in posts:
        score = ml_model.predict(user_features, post.features)
        scored_posts.append((post, score))

    return [p for p, s in sorted(scored_posts, key=lambda x: -x[1])]
```

## 5. Database Design

### Posts Table

```sql
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    caption TEXT,
    location VARCHAR(255),
    media_type VARCHAR(20),  -- photo, video, carousel
    status VARCHAR(20),
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);

-- Media URLs stored separately for flexibility
CREATE TABLE post_media (
    media_id BIGINT PRIMARY KEY,
    post_id BIGINT NOT NULL,
    media_order INT,
    original_url TEXT,
    thumb_url TEXT,
    small_url TEXT,
    medium_url TEXT,
    large_url TEXT,
    width INT,
    height INT,
    FOREIGN KEY (post_id) REFERENCES posts(post_id)
);
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

CREATE TABLE likes (
    user_id BIGINT,
    post_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id),
    INDEX idx_post_id (post_id)
);
```

### Feed Cache (Redis)

```
Key: feed:{user_id}
Value: Sorted Set of post_ids by timestamp

ZADD feed:123 1640000000 post_789
ZREVRANGE feed:123 0 20  -- Latest 20 posts
```

## 6. Stories Feature

```
┌────────────────────────────────────────────────────────┐
│                    Stories                             │
│                                                        │
│  24-hour ephemeral content                            │
│                                                        │
│  Storage:                                              │
│  - Separate bucket with lifecycle policy             │
│  - Auto-delete after 24 hours                        │
│                                                        │
│  Database:                                             │
│  CREATE TABLE stories (                               │
│      story_id BIGINT PRIMARY KEY,                    │
│      user_id BIGINT NOT NULL,                        │
│      media_url TEXT,                                  │
│      created_at TIMESTAMP,                           │
│      expires_at TIMESTAMP,                           │
│      view_count INT DEFAULT 0                        │
│  );                                                    │
│                                                        │
│  TTL-based cleanup:                                   │
│  - Redis: EXPIRE story:123 86400                     │
│  - S3: Lifecycle rule deletes after 24h             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 7. CDN Architecture

```
┌────────────────────────────────────────────────────────┐
│                   CDN Strategy                         │
│                                                        │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐      │
│  │  User    │────▶│   CDN    │────▶│ Origin   │      │
│  │(Mumbai)  │     │(Mumbai   │     │   S3     │      │
│  └──────────┘     │ Edge)    │     └──────────┘      │
│                   └──────────┘                        │
│                                                        │
│  URL format:                                           │
│  https://cdn.instagram.com/photos/{photo_id}/medium  │
│                                                        │
│  Benefits:                                             │
│  - Low latency (edge server close to user)           │
│  - Reduced S3 bandwidth costs                        │
│  - 90%+ cache hit ratio for popular content         │
│                                                        │
│  Cache headers:                                        │
│  Cache-Control: public, max-age=31536000            │
│  (Images are immutable, cache forever)               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 8. Search

```
┌────────────────────────────────────────────────────────┐
│                    Search                              │
│                                                        │
│  Elasticsearch indices:                               │
│                                                        │
│  1. Users index:                                      │
│     - username (autocomplete)                        │
│     - full_name                                       │
│     - bio                                             │
│     - follower_count (for ranking)                   │
│                                                        │
│  2. Hashtags index:                                   │
│     - tag name                                         │
│     - post_count                                       │
│                                                        │
│  3. Locations index:                                  │
│     - location name                                   │
│     - coordinates                                     │
│                                                        │
│  Search ranking:                                       │
│  - Relevance score                                    │
│  - Popularity (followers, post count)                │
│  - User's connection strength                        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Separate storage tiers** - hot data in cache, cold in S3
2. **CDN for images** - reduces latency and origin load
3. **Image processing pipeline** - async thumbnail generation
4. **Hybrid feed generation** - balance write and read costs
5. **ML-based ranking** - personalized feed ordering
6. **Stories with TTL** - lifecycle policies for cleanup

---

Next: [WhatsApp](../03-whatsapp/README.md)
