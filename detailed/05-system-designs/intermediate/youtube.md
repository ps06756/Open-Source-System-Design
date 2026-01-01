# Design YouTube

## 1. Requirements

### Functional Requirements
- Upload videos
- Watch videos (streaming)
- Search videos
- Like/dislike, comment, subscribe
- Recommendations
- Video analytics

### Non-Functional Requirements
- High availability
- Low latency for video playback
- Support multiple resolutions
- Global access
- Handle viral videos

### Scale Estimates
- 2B monthly active users
- 500M daily active users
- 500 hours of video uploaded per minute
- Average video: 10 minutes, 100 MB

```
┌─────────────────────────────────────────────────────────────┐
│                   Scale Summary                             │
├─────────────────────────────────────────────────────────────┤
│  Video uploads: 500 hours/minute = 720K hours/day          │
│  Storage: 500 × 60 × 100MB = 3 TB/minute                  │
│  Video views: 5B/day                                        │
│  Bandwidth: Massive (petabytes/day)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                        CDN                             │  │
│  │           (Video delivery to users)                    │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────┴──────────────────────────────┐  │
│  │                   Load Balancer                        │  │
│  └────────────────────────┬──────────────────────────────┘  │
│                           │                                  │
│     ┌─────────────────────┼─────────────────────┐           │
│     ▼                     ▼                     ▼           │
│ ┌────────────┐     ┌────────────┐      ┌────────────┐      │
│ │  Upload    │     │  Watch     │      │   Search   │      │
│ │  Service   │     │  Service   │      │  Service   │      │
│ └─────┬──────┘     └─────┬──────┘      └─────┬──────┘      │
│       │                  │                   │              │
│       ▼                  ▼                   ▼              │
│ ┌───────────────────────────────────────────────────────┐  │
│ │             Video Processing Pipeline                  │  │
│ │  (Transcode → Generate thumbnails → Extract metadata) │  │
│ └───────────────────────────────────────────────────────┘  │
│                          │                                  │
│            ┌─────────────┼─────────────┐                   │
│            ▼             ▼             ▼                   │
│     ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│     │ Original │  │Transcoded│  │ Metadata │              │
│     │  Storage │  │  Videos  │  │    DB    │              │
│     │   (S3)   │  │   (S3)   │  │  (MySQL) │              │
│     └──────────┘  └──────────┘  └──────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Video Upload & Processing

```
┌─────────────────────────────────────────────────────────────┐
│              Video Upload & Processing                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Upload Flow:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Client requests upload URL                         │ │
│  │  ┌────────┐   ┌───────────┐   ┌──────────────────┐    │ │
│  │  │ Client │──▶│ API Server│──▶│ Generate presigned│   │ │
│  │  └────────┘   └───────────┘   │ URL for S3       │    │ │
│  │                               └──────────────────┘    │ │
│  │                                                         │ │
│  │  2. Client uploads directly to S3                      │ │
│  │  ┌────────┐        ┌───────────────────┐              │ │
│  │  │ Client │───────▶│ S3 (original bucket)│            │ │
│  │  └────────┘        └─────────┬─────────┘              │ │
│  │                              │                         │ │
│  │  3. S3 triggers processing pipeline                   │ │
│  │                              ▼                         │ │
│  │  ┌───────────────────────────────────────────────┐    │ │
│  │  │           Video Processing Queue              │    │ │
│  │  │                  (SQS/Kafka)                  │    │ │
│  │  └───────────────────────────────────────────────┘    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Processing Pipeline:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐│ │
│  │  │  Decode  │─▶│Transcode │─▶│ Generate │─▶│ Upload ││ │
│  │  │ Original │  │ Multiple │  │Thumbnails│  │ to CDN ││ │
│  │  │          │  │Resolutions│ │          │  │        ││ │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘│ │
│  │       │             │             │                    │ │
│  │       ▼             ▼             ▼                    │ │
│  │  Extract       240p, 360p    Thumbnail at             │ │
│  │  metadata      480p, 720p    multiple points          │ │
│  │  (duration,    1080p, 4K     + preview sprites        │ │
│  │  codec, etc.)                                         │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Encoding formats:                                          │
│  • H.264 (most compatible)                                 │
│  • VP9 (better compression, WebM)                          │
│  • AV1 (newest, best compression)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Video Streaming

```
┌─────────────────────────────────────────────────────────────┐
│                   Video Streaming                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Adaptive Bitrate Streaming (ABR):                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Video segmented into small chunks (2-10 seconds)      │ │
│  │                                                         │ │
│  │  ┌───────────────────────────────────────────────────┐│ │
│  │  │     video_chunk_001.mp4  (2 seconds)              ││ │
│  │  │     video_chunk_002.mp4  (2 seconds)              ││ │
│  │  │     video_chunk_003.mp4  (2 seconds)              ││ │
│  │  │     ...                                            ││ │
│  │  └───────────────────────────────────────────────────┘│ │
│  │                                                         │ │
│  │  Manifest file (HLS/DASH):                             │ │
│  │  ┌───────────────────────────────────────────────────┐│ │
│  │  │  #EXTM3U                                          ││ │
│  │  │  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=480││ │
│  │  │  480p/playlist.m3u8                               ││ │
│  │  │  #EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=720│ │
│  │  │  720p/playlist.m3u8                               ││ │
│  │  │  #EXT-X-STREAM-INF:BANDWIDTH=4000000,RESOLUTION=1080 │
│  │  │  1080p/playlist.m3u8                              ││ │
│  │  └───────────────────────────────────────────────────┘│ │
│  │                                                         │ │
│  │  Client dynamically switches quality based on:        │ │
│  │  • Available bandwidth                                 │ │
│  │  • Buffer status                                       │ │
│  │  • Device capabilities                                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Streaming protocols:                                       │
│  • HLS (HTTP Live Streaming) - Apple                       │
│  • DASH (Dynamic Adaptive Streaming over HTTP) - Standard │
│  • Both work over HTTP/CDN                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### CDN Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CDN Architecture                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Multi-tier CDN:                                            │
│                                                              │
│  ┌────────┐                                                 │
│  │ Viewer │                                                 │
│  └───┬────┘                                                 │
│      │                                                       │
│      ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Edge PoP (Point of Presence)                          ││
│  │  • Closest to user                                     ││
│  │  • Cache popular videos                                ││
│  │  • ~200 locations globally                             ││
│  └──────────────────────────┬──────────────────────────────┘│
│                             │ Cache miss                    │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Regional PoP                                          ││
│  │  • Larger cache                                        ││
│  │  • Serves multiple edge PoPs                          ││
│  └──────────────────────────┬──────────────────────────────┘│
│                             │ Cache miss                    │
│                             ▼                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Origin (S3)                                           ││
│  │  • All video files                                     ││
│  │  • Rarely accessed directly                           ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Hot vs Cold videos:                                        │
│  • Hot (popular): Cached at edge, instant start            │
│  • Cold (rare): Fetched from origin, slightly slower start│
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
│  Videos (MySQL + Elasticsearch)                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ video_id (PK)     │ BIGINT                              │ │
│  │ user_id           │ BIGINT (FK)                         │ │
│  │ title             │ VARCHAR(100)                        │ │
│  │ description       │ TEXT                                │ │
│  │ upload_status     │ ENUM(processing, ready, failed)    │ │
│  │ duration_seconds  │ INT                                 │ │
│  │ view_count        │ BIGINT                              │ │
│  │ like_count        │ BIGINT                              │ │
│  │ thumbnail_url     │ VARCHAR(500)                        │ │
│  │ created_at        │ TIMESTAMP                           │ │
│  │ published_at      │ TIMESTAMP                           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Video Files (S3 + metadata in DB)                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ video_id          │ BIGINT (FK)                         │ │
│  │ resolution        │ ENUM(240p, 360p, 480p, 720p, 1080p)│ │
│  │ codec             │ VARCHAR(20)                         │ │
│  │ file_size_bytes   │ BIGINT                              │ │
│  │ bitrate           │ INT                                 │ │
│  │ s3_path           │ VARCHAR(500)                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  View Counts (Cassandra - high write throughput)           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ video_id          │ BIGINT (partition key)             │ │
│  │ timestamp_hour    │ TIMESTAMP (clustering key)         │ │
│  │ view_count        │ COUNTER                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Recommendations

```
┌─────────────────────────────────────────────────────────────┐
│                   Recommendations                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Recommendation Pipeline:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Candidate Generation                               │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ Sources:                                     │   │ │
│  │     │ • Subscribed channels                        │   │ │
│  │     │ • Similar to watched videos                  │   │ │
│  │     │ • Trending in region                         │   │ │
│  │     │ • Collaborative filtering                    │   │ │
│  │     │ → Generates ~1000 candidates                 │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                          │                              │ │
│  │                          ▼                              │ │
│  │  2. Ranking (ML Model)                                 │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ Features:                                    │   │ │
│  │     │ • Video features (title, category, length)  │   │ │
│  │     │ • User features (watch history, prefs)      │   │ │
│  │     │ • Context (time of day, device)             │   │ │
│  │     │ • Engagement predictions (CTR, watch time)  │   │ │
│  │     │ → Scores and ranks candidates               │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                          │                              │ │
│  │                          ▼                              │ │
│  │  3. Post-processing                                    │ │
│  │     ┌─────────────────────────────────────────────┐   │ │
│  │     │ • Diversity (mix topics/creators)           │   │ │
│  │     │ • Freshness (include new content)           │   │ │
│  │     │ • Policy (remove inappropriate)             │   │ │
│  │     │ → Final list of ~20 videos                  │   │ │
│  │     └─────────────────────────────────────────────┘   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Pre-compute for speed:                                     │
│  • User's personalized recommendations cached              │
│  • Updated periodically and on significant events          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Video Storage** | S3 | Scalable, durable |
| **Video Delivery** | Multi-tier CDN | Low latency globally |
| **Streaming** | HLS/DASH | Adaptive bitrate |
| **Metadata** | MySQL | Structured data |
| **Search** | Elasticsearch | Full-text search |
| **View Counts** | Cassandra | High write throughput |
| **Processing** | Distributed queue | Parallel transcoding |

---

[Back to System Designs](../README.md)
