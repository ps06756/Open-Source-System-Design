# Design YouTube

Design a video streaming platform like YouTube.

## 1. Requirements

### Functional Requirements
- Upload videos
- Stream videos (adaptive bitrate)
- Search videos
- Like, comment, subscribe
- Recommendations
- Live streaming

### Non-Functional Requirements
- High availability (99.99%)
- Low latency video start (<2s)
- Global reach with low buffering
- Scale: 2B users, 1B videos, 500M DAU

### Capacity Estimation

```
Video uploads: 500 hours/minute
Storage per minute: 500 × 60 × (multiple resolutions)
                  = ~150 GB/minute = 216 TB/day

Video watches: 1B hours/day
Bandwidth: 1B × 4 Mbps avg = 4 Pbps peak

Storage (5 years):
Video: 216 TB × 365 × 5 = 400 PB
Metadata: 1B videos × 1 KB = 1 TB
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │ Client │────▶│ Load Balancer│────▶│    API Servers     │   │
│  └────────┘     └──────────────┘     └─────────┬──────────┘   │
│       │                                         │              │
│       │         ┌───────────────────────────────┼───────────┐ │
│       │         │                               │           │ │
│       │         ▼                               ▼           │ │
│       │  ┌─────────────┐           ┌─────────────────────┐ │ │
│       │  │ Video Upload│           │   Video Metadata    │ │ │
│       │  │   Service   │           │   Service           │ │ │
│       │  └──────┬──────┘           └─────────────────────┘ │ │
│       │         │                                           │ │
│       │         ▼                                           │ │
│       │  ┌─────────────┐     ┌─────────────────────────┐   │ │
│       │  │   Encoder   │────▶│     Object Storage      │   │ │
│       │  │   Service   │     │   (Video Chunks)        │   │ │
│       │  └─────────────┘     └───────────┬─────────────┘   │ │
│       │                                   │                  │ │
│       │                                   ▼                  │ │
│       └──────────────────────────▶┌─────────────┐           │ │
│             Video Stream          │     CDN     │           │ │
│                                   └─────────────┘           │ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Video Upload Pipeline

```
┌────────────────────────────────────────────────────────────────┐
│                    Video Upload Pipeline                       │
│                                                                │
│  ┌───────────────────────────────────────────────────────────┐│
│  │  1. Upload                                                 ││
│  │     Client ──▶ Presigned URL ──▶ S3 (Original)           ││
│  └───────────────────────────────────────────────────────────┘│
│                            │                                   │
│                            ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐│
│  │  2. Transcode (Parallel)                                  ││
│  │     ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐          ││
│  │     │ 360p   │ │ 720p   │ │ 1080p  │ │ 4K     │          ││
│  │     │ encode │ │ encode │ │ encode │ │ encode │          ││
│  │     └────────┘ └────────┘ └────────┘ └────────┘          ││
│  └───────────────────────────────────────────────────────────┘│
│                            │                                   │
│                            ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐│
│  │  3. Segment (HLS/DASH)                                    ││
│  │     Split each resolution into 4-10 second chunks        ││
│  │     Generate manifest files (.m3u8 / .mpd)               ││
│  └───────────────────────────────────────────────────────────┘│
│                            │                                   │
│                            ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐│
│  │  4. Store & Distribute                                    ││
│  │     Upload chunks to S3 ──▶ Replicate to CDN edges       ││
│  └───────────────────────────────────────────────────────────┘│
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Encoding Service

```python
class VideoEncoder:
    def __init__(self):
        self.resolutions = [
            {'name': '360p', 'width': 640, 'height': 360, 'bitrate': '800k'},
            {'name': '720p', 'width': 1280, 'height': 720, 'bitrate': '2500k'},
            {'name': '1080p', 'width': 1920, 'height': 1080, 'bitrate': '5000k'},
            {'name': '4k', 'width': 3840, 'height': 2160, 'bitrate': '20000k'},
        ]

    async def process_video(self, video_id, source_path):
        """Process uploaded video"""
        # 1. Validate and extract metadata
        metadata = await self.extract_metadata(source_path)

        # 2. Generate thumbnail
        thumbnail = await self.generate_thumbnail(source_path)

        # 3. Transcode to multiple resolutions (parallel)
        tasks = []
        for res in self.resolutions:
            if res['height'] <= metadata['height']:
                task = self.transcode(source_path, video_id, res)
                tasks.append(task)

        encoded_files = await asyncio.gather(*tasks)

        # 4. Segment for streaming
        for encoded in encoded_files:
            await self.segment_video(encoded, video_id)

        # 5. Generate manifest
        manifest = await self.generate_manifest(video_id)

        # 6. Upload to CDN
        await self.upload_to_cdn(video_id)

        # 7. Update database
        await self.mark_ready(video_id)

    async def segment_video(self, video_path, video_id):
        """Split video into HLS segments"""
        # Using ffmpeg
        cmd = f"""
        ffmpeg -i {video_path} \
            -hls_time 10 \
            -hls_segment_filename '{video_id}_%03d.ts' \
            -hls_playlist_type vod \
            {video_id}.m3u8
        """
        await run_command(cmd)
```

## 4. Video Streaming

### Adaptive Bitrate Streaming (ABR)

```
┌────────────────────────────────────────────────────────────────┐
│              Adaptive Bitrate Streaming                        │
│                                                                │
│  Master Playlist (video_123.m3u8):                            │
│  #EXTM3U                                                       │
│  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360        │
│  360p/playlist.m3u8                                            │
│  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720      │
│  720p/playlist.m3u8                                            │
│  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080     │
│  1080p/playlist.m3u8                                           │
│                                                                │
│  Client behavior:                                              │
│  1. Start with lowest quality (fast start)                    │
│  2. Monitor buffer and bandwidth                              │
│  3. Switch quality based on conditions                        │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Bandwidth │  360p  │  720p  │  1080p │  4K    │         │  │
│  │    ▲      │        │        │        │        │         │  │
│  │ 20Mbps   │        │        │        │  ███   │         │  │
│  │ 5Mbps    │        │        │  ███   │        │         │  │
│  │ 2.5Mbps  │        │  ███   │        │        │         │  │
│  │ 0.8Mbps  │  ███   │        │        │        │         │  │
│  │          └────────┴────────┴────────┴────────┴──▶ Time  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 5. CDN Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    CDN Architecture                            │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │                     Origin (S3)                           │ │
│  │          All video chunks and manifests                  │ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │                                    │
│           ┌───────────────┼───────────────┐                   │
│           │               │               │                   │
│           ▼               ▼               ▼                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │ Edge (US)   │  │ Edge (EU)   │  │ Edge (Asia) │           │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │           │
│  │ │ Cache   │ │  │ │ Cache   │ │  │ │ Cache   │ │           │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │           │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │
│         │                │                │                   │
│         ▼                ▼                ▼                   │
│      Users US         Users EU        Users Asia              │
│                                                                │
│  Cache Strategy:                                               │
│  - Popular videos: Keep at all edges                          │
│  - Long-tail: Pull from origin on demand                     │
│  - TTL: 1 year (video chunks are immutable)                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Database Design

### Video Metadata

```sql
CREATE TABLE videos (
    video_id UUID PRIMARY KEY,
    title VARCHAR(255),
    description TEXT,
    channel_id UUID,
    duration_seconds INT,
    upload_status VARCHAR(20),  -- processing, ready, failed
    visibility VARCHAR(20),      -- public, private, unlisted
    view_count BIGINT DEFAULT 0,
    like_count BIGINT DEFAULT 0,
    created_at TIMESTAMP,
    published_at TIMESTAMP
);

CREATE TABLE video_resolutions (
    video_id UUID,
    resolution VARCHAR(10),  -- 360p, 720p, 1080p, 4k
    manifest_url TEXT,
    bitrate INT,
    PRIMARY KEY (video_id, resolution)
);

-- Denormalized for read performance
CREATE TABLE video_thumbnails (
    video_id UUID PRIMARY KEY,
    thumbnail_default TEXT,
    thumbnail_medium TEXT,
    thumbnail_high TEXT
);
```

### View Counting

```
┌────────────────────────────────────────────────────────────────┐
│                   View Count Architecture                      │
│                                                                │
│  Problem: Millions of concurrent views, can't update DB each  │
│                                                                │
│  Solution: Multi-tier aggregation                             │
│                                                                │
│  1. Client watches video                                      │
│  2. View event → Kafka                                        │
│  3. Spark Streaming aggregates per minute                    │
│  4. Write aggregated counts to Cassandra                     │
│  5. Periodic sync to main DB                                  │
│                                                                │
│  ┌─────────┐    ┌─────────┐    ┌─────────────┐               │
│  │ Client  │───▶│ Kafka   │───▶│ Spark       │               │
│  │ View    │    │ Topic   │    │ Streaming   │               │
│  └─────────┘    └─────────┘    └──────┬──────┘               │
│                                       │                       │
│                                       ▼                       │
│                               ┌───────────────┐               │
│                               │  Cassandra    │               │
│                               │ (Real-time)   │               │
│                               └───────┬───────┘               │
│                                       │ Hourly                │
│                                       ▼                       │
│                               ┌───────────────┐               │
│                               │  PostgreSQL   │               │
│                               │ (Source of    │               │
│                               │  truth)       │               │
│                               └───────────────┘               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 7. Recommendations

```python
class RecommendationEngine:
    def get_recommendations(self, user_id, video_id=None, limit=20):
        """Get personalized video recommendations"""

        # 1. Candidate generation (millions → thousands)
        candidates = self.get_candidates(user_id)

        # 2. Ranking (thousands → tens)
        scored = []
        for video in candidates:
            features = self.extract_features(user_id, video)
            score = self.ranking_model.predict(features)
            scored.append((video, score))

        # 3. Re-ranking for diversity
        diverse = self.diversify(scored)

        # 4. Filter unwatched and appropriate
        filtered = self.filter(diverse, user_id)

        return filtered[:limit]

    def get_candidates(self, user_id):
        """Generate candidate videos"""
        candidates = set()

        # Similar to videos user watched
        watch_history = self.get_watch_history(user_id)
        for video in watch_history[-20:]:
            similar = self.get_similar_videos(video, limit=50)
            candidates.update(similar)

        # From subscribed channels
        channels = self.get_subscribed_channels(user_id)
        for channel in channels:
            recent = self.get_channel_videos(channel, limit=10)
            candidates.update(recent)

        # Trending videos
        trending = self.get_trending(limit=100)
        candidates.update(trending)

        return list(candidates)

    def extract_features(self, user_id, video):
        """Extract features for ranking model"""
        return {
            'video_age_hours': hours_since(video.published_at),
            'video_duration': video.duration,
            'video_view_count': log(video.views),
            'channel_subscriber_count': log(video.channel.subscribers),
            'user_watch_time_channel': self.get_watch_time(user_id, video.channel),
            'user_avg_watch_duration': self.get_avg_duration(user_id),
            'category_affinity': self.get_category_affinity(user_id, video.category),
            # ... many more features
        }
```

## 8. Live Streaming

```
┌────────────────────────────────────────────────────────────────┐
│                    Live Streaming                              │
│                                                                │
│  Broadcaster ──RTMP──▶ Ingest Server ──▶ Transcoder           │
│                                              │                 │
│                              ┌───────────────┼───────────────┐│
│                              │               │               ││
│                              ▼               ▼               ▼│
│                         ┌────────┐     ┌────────┐     ┌────────┐
│                         │ 360p   │     │ 720p   │     │ 1080p  │
│                         └───┬────┘     └───┬────┘     └───┬────┘
│                             │              │              │    │
│                             └──────────────┼──────────────┘    │
│                                           │                    │
│                                           ▼                    │
│                                    ┌─────────────┐            │
│                                    │     CDN     │            │
│                                    │  (LL-HLS)   │            │
│                                    └──────┬──────┘            │
│                                           │                    │
│                                           ▼                    │
│                                       Viewers                  │
│                                                                │
│  Low-Latency HLS (LL-HLS):                                   │
│  - 2-5 second latency                                        │
│  - Partial segments (0.2s chunks)                            │
│  - Preload hints                                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 9. Key Takeaways

1. **Parallel transcoding** - encode multiple resolutions simultaneously
2. **Adaptive bitrate** - seamless quality switching based on bandwidth
3. **CDN with edge caching** - reduce latency and origin load
4. **Async view counting** - aggregate to handle high volume
5. **ML recommendations** - two-stage: candidates → ranking
6. **HLS/DASH** - industry standard for streaming

---

Next: [Uber](../02-uber/README.md)
