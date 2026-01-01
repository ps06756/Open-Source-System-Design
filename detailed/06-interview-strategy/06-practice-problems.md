# Practice Problems

Self-assessment exercises to test your system design skills.

---

## How to Use This Section

```
┌─────────────────────────────────────────────────────────────┐
│                Practice Instructions                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  For each problem:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Set a timer for 45 minutes                         │ │
│  │  2. Read only the problem statement (not hints)        │ │
│  │  3. Design on paper or whiteboard                      │ │
│  │  4. After time's up, compare with key points           │ │
│  │  5. Rate yourself on the rubric                        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Scoring (per problem):                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements (20 points)                               │ │
│  │  • Clarified scope: 5 pts                              │ │
│  │  • Estimated scale: 5 pts                              │ │
│  │  • Identified key features: 10 pts                     │ │
│  │                                                         │ │
│  │  High-Level Design (30 points)                         │ │
│  │  • Clear architecture: 15 pts                          │ │
│  │  • Data flow shown: 10 pts                             │ │
│  │  • Key components identified: 5 pts                    │ │
│  │                                                         │ │
│  │  Deep Dive (30 points)                                  │ │
│  │  • Database design: 10 pts                             │ │
│  │  • Scaling strategy: 10 pts                            │ │
│  │  • Trade-offs discussed: 10 pts                        │ │
│  │                                                         │ │
│  │  Extras (20 points)                                     │ │
│  │  • Failure handling: 10 pts                            │ │
│  │  • Monitoring: 5 pts                                   │ │
│  │  • Improvements: 5 pts                                 │ │
│  │                                                         │ │
│  │  Total: 100 points                                      │ │
│  │  70+: Ready for interviews                             │ │
│  │  50-69: Need more practice                             │ │
│  │  <50: Review fundamentals                              │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 1: Design a URL Shortener (Beginner)

```
┌─────────────────────────────────────────────────────────────┐
│                  URL Shortener                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a URL shortening service like bit.ly.          │ │
│  │                                                         │ │
│  │  Users should be able to:                              │ │
│  │  • Create short URLs from long URLs                    │ │
│  │  • Redirect short URLs to original URLs                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 100M URLs created per month                         │ │
│  │  □ 100:1 read/write ratio                              │ │
│  │  □ URL expiration (optional)                           │ │
│  │  □ Analytics (optional)                                │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Short code generation (Base62 encoding)             │ │
│  │  □ 7 characters = 62^7 = 3.5 trillion combinations     │ │
│  │  □ Database choice (SQL for durability)                │ │
│  │  □ Caching for hot URLs                                │ │
│  │                                                         │ │
│  │  Scaling:                                               │ │
│  │  □ Write QPS: ~40 (easy)                               │ │
│  │  □ Read QPS: ~4,000 (cache helps)                      │ │
│  │  □ Storage: ~600GB over 5 years                        │ │
│  │                                                         │ │
│  │  Trade-offs:                                            │ │
│  │  □ Counter vs hash for ID generation                   │ │
│  │  □ Collision handling                                  │ │
│  │  □ Custom aliases                                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 2: Design Twitter Timeline (Intermediate)

```
┌─────────────────────────────────────────────────────────────┐
│                  Twitter Timeline                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design the home timeline feature for Twitter.         │ │
│  │                                                         │ │
│  │  Users should be able to:                              │ │
│  │  • Post tweets                                         │ │
│  │  • View their home timeline                            │ │
│  │  • Follow other users                                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 200M DAU, 500M tweets/day                           │ │
│  │  □ Average user follows 200 people                     │ │
│  │  □ Some users (celebrities) have millions followers    │ │
│  │  □ Timeline should load in < 200ms                     │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Fan-out on write vs fan-out on read                 │ │
│  │  □ Hybrid approach for celebrities                     │ │
│  │  □ Pre-computed timeline cache                         │ │
│  │  □ Pagination strategy (cursor-based)                  │ │
│  │                                                         │ │
│  │  The Celebrity Problem:                                 │ │
│  │  □ Celebrities don't fan-out (too expensive)           │ │
│  │  □ Mix in celebrity tweets at read time                │ │
│  │  □ Cache celebrity tweets separately                   │ │
│  │                                                         │ │
│  │  Data Model:                                            │ │
│  │  □ User table, Tweet table, Follow table               │ │
│  │  □ Timeline cache (user_id → tweet_ids)                │ │
│  │  □ Sharding strategy                                   │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 3: Design a Chat System (Intermediate)

```
┌─────────────────────────────────────────────────────────────┐
│                    Chat System                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a real-time chat application like WhatsApp.    │ │
│  │                                                         │ │
│  │  Users should be able to:                              │ │
│  │  • Send messages to other users (1:1)                  │ │
│  │  • See message delivery status                         │ │
│  │  • View online status                                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │ │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 50M DAU, 20 messages sent per user per day          │ │
│  │  □ Real-time delivery (< 100ms)                        │ │
│  │  □ Messages stored forever                             │ │
│  │  □ Handle offline users                                │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ WebSocket for real-time                             │ │
│  │  □ Connection server for WebSocket management          │ │
│  │  □ Message queue for async delivery                    │ │
│  │  □ Delivery receipts (sent, delivered, read)           │ │
│  │                                                         │ │
│  │  Handling Offline Users:                                │ │
│  │  □ Store in message queue                              │ │
│  │  □ Push notification                                   │ │
│  │  □ Sync on reconnect                                   │ │
│  │                                                         │ │
│  │  Data Model:                                            │ │
│  │  □ Message: id, sender, receiver, content, timestamp   │ │
│  │  □ Conversation: last_message, unread_count            │ │
│  │  □ Online status: last_active_time                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 4: Design a Ride-Sharing Service (Intermediate)

```
┌─────────────────────────────────────────────────────────────┐
│                 Ride-Sharing Service                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a ride-sharing service like Uber/Lyft.         │ │
│  │                                                         │ │
│  │  Users should be able to:                              │ │
│  │  • Request a ride                                      │ │
│  │  • Get matched with nearby drivers                     │ │
│  │  • Track driver location                               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 10M daily rides, 1M concurrent drivers              │ │
│  │  □ Match within 30 seconds                             │ │
│  │  □ Driver location updates every 3 seconds             │ │
│  │  □ ETA accuracy within 2 minutes                       │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Geospatial indexing (Geohash, Quadtree)             │ │
│  │  □ Location update handling (high write throughput)    │ │
│  │  □ Matching algorithm                                  │ │
│  │  □ ETA calculation                                     │ │
│  │                                                         │ │
│  │  Location Tracking:                                     │ │
│  │  □ 1M drivers × 1 update/3s = 333K writes/second       │ │
│  │  □ Need in-memory store (Redis with geo commands)      │ │
│  │  □ Periodic persistence to database                    │ │
│  │                                                         │ │
│  │  Matching:                                              │ │
│  │  □ Find drivers in expanding radius                    │ │
│  │  □ Consider ETA, not just distance                     │ │
│  │  □ Handle supply/demand imbalance                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 5: Design a Video Streaming Service (Advanced)

```
┌─────────────────────────────────────────────────────────────┐
│              Video Streaming Service                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a video streaming service like Netflix.        │ │
│  │                                                         │ │
│  │  Users should be able to:                              │ │
│  │  • Browse and search for videos                        │ │
│  │  • Stream videos at various quality levels             │ │
│  │  • Resume where they left off                          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 200M subscribers, 50M concurrent viewers            │ │
│  │  □ 10,000 titles, 100+ hours added daily               │ │
│  │  □ Support 4K, 1080p, 720p, 480p, 360p                 │ │
│  │  □ Buffer-free playback                                │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Video encoding pipeline                             │ │
│  │  □ Adaptive bitrate streaming (HLS/DASH)               │ │
│  │  □ CDN for video delivery                              │ │
│  │  □ Video chunk size (2-10 seconds)                     │ │
│  │                                                         │ │
│  │  Storage:                                               │ │
│  │  □ Original + multiple resolutions                     │ │
│  │  □ Object storage (S3)                                 │ │
│  │  □ CDN caching strategy                                │ │
│  │                                                         │ │
│  │  Bandwidth:                                             │ │
│  │  □ 50M viewers × 5 Mbps avg = 250 Tbps                 │ │
│  │  □ CDN is essential (can't serve from origin)          │ │
│  │  □ Predictive pre-positioning of content               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 6: Design a Distributed Cache (Advanced)

```
┌─────────────────────────────────────────────────────────────┐
│                 Distributed Cache                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a distributed caching system like Memcached.   │ │
│  │                                                         │ │
│  │  The system should:                                     │ │
│  │  • Support get/set/delete operations                   │ │
│  │  • Scale horizontally                                  │ │
│  │  • Handle node failures                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 1M QPS, sub-millisecond latency                     │ │
│  │  □ 1TB total cache size                                │ │
│  │  □ High availability (99.99%)                          │ │
│  │  □ Handle hot keys                                     │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Consistent hashing for distribution                 │ │
│  │  □ Virtual nodes for balance                           │ │
│  │  □ Replication strategy                                │ │
│  │  □ Eviction policy (LRU, LFU)                          │ │
│  │                                                         │ │
│  │  Hot Key Handling:                                      │ │
│  │  □ Local cache (L1)                                    │ │
│  │  □ Key replication across nodes                        │ │
│  │  □ Load-aware routing                                  │ │
│  │                                                         │ │
│  │  Failure Handling:                                      │ │
│  │  □ Replication factor (typically 3)                    │ │
│  │  □ Failover mechanism                                  │ │
│  │  □ Data reconstruction                                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem 7: Design a Rate Limiter (Advanced)

```
┌─────────────────────────────────────────────────────────────┐
│                   Rate Limiter                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM STATEMENT:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Design a rate limiting service for an API gateway.    │ │
│  │                                                         │ │
│  │  The system should:                                     │ │
│  │  • Limit requests per user/API key                     │ │
│  │  • Support multiple rate limit rules                   │ │
│  │  • Work across distributed servers                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ─────────────── STOP HERE AND DESIGN ───────────────────  │
│                                                              │
│  KEY POINTS TO COVER (check after designing):              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Requirements:                                          │ │
│  │  □ 100K+ rate limit checks per second                  │ │
│  │  □ Low latency (< 1ms additional)                      │ │
│  │  □ Multiple dimensions (user, IP, API)                 │ │
│  │  □ Configurable limits                                 │ │
│  │                                                         │ │
│  │  Key Design Decisions:                                  │ │
│  │  □ Algorithm choice (token bucket, sliding window)     │ │
│  │  □ Centralized vs distributed                          │ │
│  │  □ Storage (Redis for distributed)                     │ │
│  │  □ Handling race conditions                            │ │
│  │                                                         │ │
│  │  Algorithms:                                            │ │
│  │  □ Token bucket: Smooth, allows bursts                 │ │
│  │  □ Sliding window: Accurate, more complex              │ │
│  │  □ Fixed window: Simple, edge case issues              │ │
│  │                                                         │ │
│  │  Distributed Considerations:                            │ │
│  │  □ Redis INCR with EXPIRE                              │ │
│  │  □ Lua scripts for atomicity                           │ │
│  │  □ Local cache for hot users                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Self-Assessment Template

```
┌─────────────────────────────────────────────────────────────┐
│              Post-Practice Assessment                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: ___________________                               │
│  Time Spent: ____ minutes                                   │
│                                                              │
│  Requirements Phase:           /20                          │
│  □ Asked about scale           /5                           │
│  □ Estimated numbers           /5                           │
│  □ Identified key features     /10                          │
│                                                              │
│  High-Level Design:            /30                          │
│  □ Clear architecture          /15                          │
│  □ Data flow shown             /10                          │
│  □ Components identified       /5                           │
│                                                              │
│  Deep Dive:                    /30                          │
│  □ Database design             /10                          │
│  □ Scaling strategy            /10                          │
│  □ Trade-offs discussed        /10                          │
│                                                              │
│  Extras:                       /20                          │
│  □ Failure handling            /10                          │
│  □ Monitoring mentioned        /5                           │
│  □ Improvements listed         /5                           │
│                                                              │
│  TOTAL:                        /100                         │
│                                                              │
│  What went well:                                            │
│  _________________________________________________         │
│                                                              │
│  What to improve:                                           │
│  _________________________________________________         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Back to Module](./README.md) | [Next: Mock Interview Guide →](./07-mock-interview-guide.md)
