# Design Netflix

## 1. Requirements

### Functional Requirements
- Stream video content
- Search and browse catalog
- Personalized recommendations
- Multiple profiles per account
- Download for offline viewing
- Continue watching across devices

### Non-Functional Requirements
- High availability (99.99%)
- Low latency for video start (< 2 seconds)
- Support millions of concurrent streams
- Adaptive quality based on bandwidth
- Global reach

### Scale Estimates
- 250M subscribers
- 100M concurrent viewers at peak
- 15,000+ titles
- 140M hours watched per day

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                      CDN (OCA)                        │  │
│  │           Open Connect Appliances                     │  │
│  │          (Video delivery at edge)                     │  │
│  └────────────────────────┬─────────────────────────────┘  │
│                           │                                  │
│  ┌────────────────────────┴─────────────────────────────┐  │
│  │              AWS (Control Plane)                      │  │
│  │  ┌─────────────────────────────────────────────────┐ │  │
│  │  │                 API Gateway                      │ │  │
│  │  └─────────────────────────────────────────────────┘ │  │
│  │                         │                             │  │
│  │      ┌──────────────────┼──────────────────┐         │  │
│  │      ▼                  ▼                  ▼         │  │
│  │ ┌──────────┐     ┌──────────┐      ┌──────────┐     │  │
│  │ │  User    │     │  Content │      │  Play    │     │  │
│  │ │ Service  │     │ Discovery│      │ Service  │     │  │
│  │ └──────────┘     └──────────┘      └──────────┘     │  │
│  │                                                       │  │
│  │      ┌─────────────────────────────────────┐         │  │
│  │      ▼                                     ▼         │  │
│  │ ┌──────────────┐              ┌──────────────────┐  │  │
│  │ │  Cassandra   │              │   Recommendation │  │  │
│  │ │  (User data) │              │      Engine      │  │  │
│  │ └──────────────┘              └──────────────────┘  │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Video Encoding Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                Video Encoding Pipeline                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Original Master File                                        │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              Encoding Pipeline                          ││
│  │                                                         ││
│  │  1. Analysis                                            ││
│  │     ┌───────────────────────────────────────────────┐  ││
│  │     │ • Scene complexity analysis                    │  ││
│  │     │ • Motion detection                             │  ││
│  │     │ • Shot boundary detection                      │  ││
│  │     └───────────────────────────────────────────────┘  ││
│  │                                                         ││
│  │  2. Per-Title Encoding                                 ││
│  │     ┌───────────────────────────────────────────────┐  ││
│  │     │ Optimize bitrate ladder per title             │  ││
│  │     │                                                │  ││
│  │     │ Action movie: Higher bitrate for motion       │  ││
│  │     │ Animated: Lower bitrate (simpler scenes)      │  ││
│  │     │                                                │  ││
│  │     │ Result: Better quality at same bandwidth      │  ││
│  │     └───────────────────────────────────────────────┘  ││
│  │                                                         ││
│  │  3. Generate Encoding Ladder                           ││
│  │     ┌───────────────────────────────────────────────┐  ││
│  │     │ Resolution   │ Bitrate  │ Codec              │  ││
│  │     │ ────────────────────────────────────────────  │  ││
│  │     │ 4K (2160p)   │ 16 Mbps  │ HEVC/VP9          │  ││
│  │     │ 1080p        │ 5 Mbps   │ HEVC/VP9/AVC      │  ││
│  │     │ 720p         │ 3 Mbps   │ HEVC/VP9/AVC      │  ││
│  │     │ 480p         │ 1.5 Mbps │ AVC                │  ││
│  │     │ 360p         │ 0.5 Mbps │ AVC                │  ││
│  │     └───────────────────────────────────────────────┘  ││
│  │                                                         ││
│  │  4. Chunk and Package                                  ││
│  │     ┌───────────────────────────────────────────────┐  ││
│  │     │ • Segment into 2-4 second chunks              │  ││
│  │     │ • Create DASH/HLS manifests                   │  ││
│  │     │ • Add DRM (Widevine, PlayReady, FairPlay)    │  ││
│  │     └───────────────────────────────────────────────┘  ││
│  │                                                         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Output: Hundreds of files per title                       │
│  (Different resolutions × languages × codecs)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Content Delivery (Open Connect)

```
┌─────────────────────────────────────────────────────────────┐
│              Open Connect Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Netflix's own CDN (not using third-party CDN):            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Open Connect Appliances (OCA)                         │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │ • Hardware: Custom servers with SSDs            │  │ │
│  │  │ • Location: ISP data centers, IXPs              │  │ │
│  │  │ • Capacity: 100+ TB per appliance               │  │ │
│  │  │ • Quantity: 17,000+ worldwide                   │  │ │
│  │  │ • Free to ISPs (Netflix provides hardware)     │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  │  Why ISP-located?                                      │ │
│  │  • Reduces ISP backbone traffic                       │ │
│  │  • Lower latency for users                            │ │
│  │  • Better streaming quality                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Content Distribution:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌───────────┐                                         │ │
│  │  │  AWS S3   │  Origin (master copies)                │ │
│  │  └─────┬─────┘                                         │ │
│  │        │                                                │ │
│  │        │  Fill during off-peak hours                   │ │
│  │        ▼                                                │ │
│  │  ┌───────────────────────────────────────────────┐    │ │
│  │  │            OCAs at ISPs                        │    │ │
│  │  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐         │    │ │
│  │  │  │OCA 1│  │OCA 2│  │OCA 3│  │OCA n│         │    │ │
│  │  │  └─────┘  └─────┘  └─────┘  └─────┘         │    │ │
│  │  └───────────────────────────────────────────────┘    │ │
│  │                                                         │ │
│  │  Pre-position popular content based on predictions    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Playback Flow:                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. User clicks Play                                   │ │
│  │  2. Client → Netflix API: "I want to watch X"         │ │
│  │  3. API returns: Best OCA URLs based on:              │ │
│  │     • User location                                    │ │
│  │     • ISP                                              │ │
│  │     • OCA health/load                                 │ │
│  │  4. Client streams from nearest OCA                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Adaptive Streaming

```
┌─────────────────────────────────────────────────────────────┐
│                  Adaptive Streaming                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Client-side ABR (Adaptive Bitrate) Algorithm:             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Input signals:                                        │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │ • Measured throughput                            │  │ │
│  │  │ • Buffer level                                   │  │ │
│  │  │ • Historical bandwidth                           │  │ │
│  │  │ • Device capabilities                            │  │ │
│  │  │ • Network type (WiFi, 4G, 5G)                   │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  │  Decision:                                              │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │  if buffer_low:                                  │  │ │
│  │  │      switch_down()  # Avoid rebuffering         │  │ │
│  │  │  elif throughput_stable_high:                    │  │ │
│  │  │      switch_up()    # Improve quality           │  │ │
│  │  │  else:                                           │  │ │
│  │  │      stay_current() # Avoid oscillation         │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  │  Quality switches happen at chunk boundaries           │ │
│  │  (every 2-4 seconds)                                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Buffer Management:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  [■■■■■■■■■■░░░░░░░░░░░░░░░░░░░░]                     │ │
│  │   └─played─┘└──buffer──┘└──empty──┘                   │ │
│  │                                                         │ │
│  │  Target buffer: 30-60 seconds ahead                   │ │
│  │  Minimum buffer: 5 seconds (before rebuffer warning)  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Recommendation System

```
┌─────────────────────────────────────────────────────────────┐
│                  Recommendation System                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  80% of watched content comes from recommendations         │
│                                                              │
│  Recommendation Pipeline:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Candidate Generation (Offline)                     │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Collaborative Filtering:                       │ │ │
│  │     │ "Users like you watched..."                   │ │ │
│  │     │                                                │ │ │
│  │     │ Content-Based:                                 │ │ │
│  │     │ "Similar to what you watched..."              │ │ │
│  │     │                                                │ │ │
│  │     │ Trending:                                      │ │ │
│  │     │ "Popular right now..."                        │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  2. Ranking (Near-real-time)                           │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Features:                                      │ │ │
│  │     │ • Watch history                               │ │ │
│  │     │ • Time of day                                 │ │ │
│  │     │ • Device type                                 │ │ │
│  │     │ • Genre preferences                           │ │ │
│  │     │ • Completion rates                            │ │ │
│  │     │                                                │ │ │
│  │     │ ML Model: Predict probability of watching    │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  │  3. Artwork Personalization                            │ │
│  │     ┌───────────────────────────────────────────────┐ │ │
│  │     │ Different thumbnails for different users      │ │ │
│  │     │                                                │ │ │
│  │     │ Action fan → Action scene thumbnail           │ │ │
│  │     │ Romance fan → Romance scene thumbnail         │ │ │
│  │     │ Actor fan → That actor's thumbnail           │ │ │
│  │     └───────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  A/B Testing:                                               │
│  • Constantly testing new algorithms                       │
│  • Measure: Engagement, retention, satisfaction            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Data Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Model                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Users & Profiles (Cassandra)                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ account_id       │ UUID                                 │ │
│  │ email            │ VARCHAR                              │ │
│  │ subscription_tier│ ENUM(basic, standard, premium)      │ │
│  │ profiles         │ LIST<profile>                        │ │
│  │ created_at       │ TIMESTAMP                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Content Catalog (MySQL + Elasticsearch)                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ content_id       │ BIGINT                               │ │
│  │ title            │ VARCHAR                              │ │
│  │ type             │ ENUM(movie, series)                  │ │
│  │ genres           │ LIST<VARCHAR>                        │ │
│  │ release_year     │ INT                                  │ │
│  │ duration_minutes │ INT                                  │ │
│  │ maturity_rating  │ VARCHAR                              │ │
│  │ available_regions│ LIST<VARCHAR>                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Viewing History (Cassandra - time series)                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ profile_id       │ UUID (partition key)                 │ │
│  │ content_id       │ BIGINT                               │ │
│  │ watched_at       │ TIMESTAMP (clustering key)           │ │
│  │ position_seconds │ INT                                  │ │
│  │ completed        │ BOOLEAN                              │ │
│  │ device_type      │ VARCHAR                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Video Delivery** | Open Connect (own CDN) | ISP-level caching |
| **Streaming** | DASH + HLS | Adaptive bitrate |
| **Encoding** | Per-title optimization | Quality/bandwidth balance |
| **User Data** | Cassandra | High availability, scale |
| **Recommendations** | ML pipeline | Personalization |
| **Control Plane** | AWS | Flexibility, scale |

---

[Back to System Designs](../README.md)
