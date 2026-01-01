# Design Instagram

## 1. Requirements

### Functional Requirements
- Upload photos/videos with captions
- Follow/unfollow users
- View feed (posts from followed users)
- Like and comment on posts
- Direct messaging
- Stories (24-hour ephemeral content)
- Explore page (discover new content)

### Non-Functional Requirements
- High availability (99.9%)
- Low latency (< 200ms for feed)
- Support for high-resolution images
- Global reach with low latency
- Eventually consistent (feed can be slightly delayed)

### Scale Estimates
- 2B monthly active users
- 500M daily active users
- 100M photos uploaded daily
- Average photo: 2 MB
- 5 photos viewed per second per user

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Photo uploads: 100M/day = 1,150/second                    │
│  Photo views: 500M × 5 = 2.5B/day = 29K/second            │
│  Storage: 100M × 2MB = 200 TB/day                         │
│  Yearly storage: 73 PB (without compression)              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                      CDN                              │  │
│  │              (Images, Videos)                         │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────┴─────────────────────────────┐  │
│  │                 Load Balancer                         │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│     ┌─────────────────────┼─────────────────────┐           │
│     ▼                     ▼                     ▼           │
│ ┌────────┐          ┌──────────┐          ┌────────┐       │
│ │ Post   │          │  Feed    │          │ User   │       │
│ │Service │          │ Service  │          │Service │       │
│ └───┬────┘          └────┬─────┘          └───┬────┘       │
│     │                    │                    │             │
│     ▼                    ▼                    ▼             │
│ ┌────────┐          ┌──────────┐          ┌────────┐       │
│ │ Post   │          │   Feed   │          │ User   │       │
│ │  DB    │          │  Cache   │          │  DB    │       │
│ └───┬────┘          └──────────┘          └────────┘       │
│     │                                                       │
│     ▼                                                       │
│ ┌────────────────────────────────────────────────────────┐ │
│ │                   Object Storage (S3)                   │ │
│ │              (Original + Resized Images)                │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Photo Upload Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   Photo Upload Flow                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────┐    1. Upload request    ┌─────────────┐        │
│  │ Client │────────────────────────▶│ API Gateway │        │
│  └────────┘                         └──────┬──────┘        │
│                                            │                │
│                            2. Get pre-signed URL           │
│                                            │                │
│  ┌────────┐    3. Direct upload     ┌──────▼──────┐        │
│  │ Client │────────────────────────▶│     S3      │        │
│  └────────┘                         └──────┬──────┘        │
│                                            │                │
│                            4. Upload complete notification │
│                                            │                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Media Processing Pipeline                │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────────┐ │  │
│  │  │  Resize   │─▶│  Filter   │─▶│ Generate versions │ │  │
│  │  │ (Multiple │  │ (Optional)│  │ (thumb, small,    │ │  │
│  │  │  sizes)   │  │           │  │  medium, large)   │ │  │
│  │  └───────────┘  └───────────┘  └───────────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                 │
│                           ▼                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  5. Store metadata + Push to CDN                      │  │
│  │  6. Fan-out to followers' feeds                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  Image Versions:                                            │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Thumbnail: 150x150 px (~10 KB)                        ││
│  │  Small: 320x320 px (~50 KB)                            ││
│  │  Medium: 640x640 px (~150 KB)                          ││
│  │  Large: 1080x1080 px (~500 KB)                         ││
│  │  Original: Preserved for downloads                     ││
│  └────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Feed Generation

```
┌─────────────────────────────────────────────────────────────┐
│                   Feed Generation                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Instagram uses ML-ranked feed (not chronological):        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Feed Ranking Pipeline                                  │ │
│  │                                                         │ │
│  │  1. Candidate Generation                               │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ • Posts from followed users                  │   │ │
│  │     │ • Suggested posts (explore)                  │   │ │
│  │     │ • Ads                                        │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                          │                              │ │
│  │                          ▼                              │ │
│  │  2. Ranking Model (ML)                                 │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ Features:                                    │   │ │
│  │     │ • User-post affinity                        │   │ │
│  │     │ • Post engagement rate                      │   │ │
│  │     │ • Recency                                   │   │ │
│  │     │ • User engagement history                   │   │ │
│  │     │ • Content type preference                   │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                          │                              │ │
│  │                          ▼                              │ │
│  │  3. Final Ranked Feed                                  │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ Diversified, deduped, policy filtered       │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Caching Strategy:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Pre-compute feeds for active users                  │ │
│  │  • Cache in Redis with TTL                             │ │
│  │  • Refresh on new post or after TTL                    │ │
│  │  • Celebrity handling similar to Twitter               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Users (MySQL)                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ user_id (PK)     │ BIGINT                               │ │
│  │ username         │ VARCHAR(30) UNIQUE                   │ │
│  │ email            │ VARCHAR(255)                         │ │
│  │ bio              │ VARCHAR(150)                         │ │
│  │ profile_pic_url  │ VARCHAR(500)                         │ │
│  │ followers_count  │ INT                                  │ │
│  │ following_count  │ INT                                  │ │
│  │ posts_count      │ INT                                  │ │
│  │ is_private       │ BOOLEAN                              │ │
│  │ created_at       │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Posts (Cassandra - partitioned by user_id)                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ post_id (PK)     │ BIGINT (Snowflake ID)               │ │
│  │ user_id          │ BIGINT (partition key)              │ │
│  │ media_urls       │ LIST<MAP<resolution, url>>          │ │
│  │ caption          │ TEXT                                 │ │
│  │ location         │ TEXT                                 │ │
│  │ likes_count      │ COUNTER                              │ │
│  │ comments_count   │ COUNTER                              │ │
│  │ created_at       │ TIMESTAMP (clustering key)          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Follows (Graph DB or Cassandra)                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ follower_id      │ BIGINT                               │ │
│  │ followee_id      │ BIGINT                               │ │
│  │ created_at       │ TIMESTAMP                            │ │
│  │ PRIMARY KEY ((follower_id), followee_id)               │ │
│  │ INDEX (followee_id) -- for "followers" query           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Likes (Cassandra)                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ post_id          │ BIGINT (partition key)              │ │
│  │ user_id          │ BIGINT (clustering key)             │ │
│  │ created_at       │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Feed Cache (Redis)                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Key: feed:{user_id}                                     │ │
│  │ Value: Sorted Set of post_ids by score                 │ │
│  │ TTL: 24 hours                                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Stories

```
┌─────────────────────────────────────────────────────────────┐
│                        Stories                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Requirements:                                               │
│  • 24-hour ephemeral content                               │
│  • Fast loading (users expect instant playback)            │
│  • View tracking                                            │
│                                                              │
│  Architecture:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Upload Story                                           │ │
│  │  ┌────────┐   ┌─────────┐   ┌───────────────────────┐ │ │
│  │  │ Client │──▶│ Process │──▶│ Store in S3 + CDN    │ │ │
│  │  └────────┘   └─────────┘   └───────────────────────┘ │ │
│  │                                                         │ │
│  │  Story Data:                                            │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ story_id      │ BIGINT                            │ │ │
│  │  │ user_id       │ BIGINT                            │ │ │
│  │  │ media_url     │ VARCHAR                           │ │ │
│  │  │ media_type    │ ENUM(photo, video)               │ │ │
│  │  │ created_at    │ TIMESTAMP                         │ │ │
│  │  │ expires_at    │ TIMESTAMP (created + 24h)        │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  Story Tray (Redis):                                   │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │ Key: story_tray:{user_id}                        │ │ │
│  │  │ Value: List of user_ids with active stories     │ │ │
│  │  │ Ordered by: Most recent story first             │ │ │
│  │  │ Includes: First story thumbnail, seen status    │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  Cleanup: TTL-based expiration + background job       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. CDN and Image Optimization

```
┌─────────────────────────────────────────────────────────────┐
│              CDN and Image Optimization                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CDN Strategy:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Client ──▶ Edge Location ──▶ Origin (S3)             │ │
│  │             (cached)                                    │ │
│  │                                                         │ │
│  │  Cache headers:                                         │ │
│  │  Cache-Control: public, max-age=31536000              │ │
│  │  (1 year - images are immutable)                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Image URL format:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  https://cdn.instagram.com/{post_id}_{size}.jpg       │ │
│  │                                                         │ │
│  │  Sizes: t (thumbnail), s (small), m (medium), l (large)│ │
│  │                                                         │ │
│  │  Client selects size based on display dimensions      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Modern optimization:                                       │
│  • WebP format for smaller sizes                           │
│  • AVIF for even better compression                        │
│  • Progressive JPEG for perceived performance              │
│  • Lazy loading for off-screen images                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Image Storage** | S3 | Scalable, durable, cost-effective |
| **Image Delivery** | CDN (CloudFront) | Low latency globally |
| **Metadata** | Cassandra | Write-heavy, time-series |
| **User Data** | MySQL | ACID, relationships |
| **Feed Cache** | Redis | Fast sorted sets |
| **Feed Ranking** | ML Pipeline | Personalized experience |

---

[Back to System Designs](../README.md)
