# Requirements Gathering

The most critical skill that separates great candidates from good ones.

---

## Why Requirements Matter

```
┌─────────────────────────────────────────────────────────────┐
│           The Impact of Requirements Gathering              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without clarification:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Design Twitter"                                       │ │
│  │                                                         │ │
│  │  ❌ Candidate assumes scope                             │ │
│  │  ❌ Designs features interviewer didn't want            │ │
│  │  ❌ Misses critical requirements                        │ │
│  │  ❌ Wastes 10 minutes on wrong direction               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  With clarification:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Design Twitter"                                       │ │
│  │                                                         │ │
│  │  ✓ Candidate asks about scope                          │ │
│  │  ✓ Aligns on key features                              │ │
│  │  ✓ Understands scale requirements                      │ │
│  │  ✓ Designs exactly what's needed                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Interviewers WANT you to ask questions!                   │
│  It shows real-world experience.                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Functional Requirements

### User-Centric Questions

```
┌─────────────────────────────────────────────────────────────┐
│                User-Centric Questions                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Who are the users?                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Questions:                                             │ │
│  │  • "Who are the primary users of this system?"         │ │
│  │  • "Are there different user types with different needs?"│
│  │  • "Is this B2C, B2B, or internal?"                    │ │
│  │                                                         │ │
│  │  Why it matters:                                        │ │
│  │  • B2C: Millions of users, simple UX                   │ │
│  │  • B2B: Thousands of users, complex features           │ │
│  │  • Internal: Hundreds of users, specific workflows     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. What can users do?                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Questions:                                             │ │
│  │  • "What are the core actions users can perform?"      │ │
│  │  • "What's the most important user journey?"           │ │
│  │  • "Are there admin vs regular user capabilities?"     │ │
│  │                                                         │ │
│  │  Example (Uber):                                        │ │
│  │  • Rider: Request ride, track driver, pay              │ │
│  │  • Driver: Accept ride, navigate, complete trip        │ │
│  │  • Admin: Manage fleet, resolve disputes               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Feature Scope Questions

```
┌─────────────────────────────────────────────────────────────┐
│                 Feature Scope Questions                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Prioritization:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Given the time, which features should we focus on?"  │ │
│  │                                                         │ │
│  │  Must Have (P0):                                        │ │
│  │  • Core functionality that defines the product         │ │
│  │  • Without these, the system is useless                │ │
│  │                                                         │ │
│  │  Should Have (P1):                                      │ │
│  │  • Important but not critical                          │ │
│  │  • Can be added after core is working                  │ │
│  │                                                         │ │
│  │  Nice to Have (P2):                                     │ │
│  │  • Enhancements and optimizations                      │ │
│  │  • Mention but don't design in detail                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example - Designing WhatsApp:                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  P0 (Must Have):                                        │ │
│  │  ✓ Send/receive messages                               │ │
│  │  ✓ 1:1 conversations                                   │ │
│  │  ✓ Message delivery status                             │ │
│  │                                                         │ │
│  │  P1 (Should Have):                                      │ │
│  │  ○ Group messaging                                     │ │
│  │  ○ Media sharing (images, videos)                      │ │
│  │  ○ Online status                                       │ │
│  │                                                         │ │
│  │  P2 (Nice to Have):                                     │ │
│  │  ○ Voice/video calls                                   │ │
│  │  ○ Stories/status updates                              │ │
│  │  ○ End-to-end encryption                               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Data Questions

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Questions                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Content Type:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "What type of content will be stored?"                │ │
│  │                                                         │ │
│  │  • Text only → Simple storage                          │ │
│  │  • Images → Object storage, CDN                        │ │
│  │  • Videos → Encoding, streaming                        │ │
│  │  • Mixed → Different storage strategies                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Retention:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "How long should we keep the data?"                   │ │
│  │                                                         │ │
│  │  • Forever → Need archival strategy                    │ │
│  │  • 7 days → Automatic cleanup                          │ │
│  │  • User-controlled → Soft deletes                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Access Patterns:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "How will data be accessed?"                          │ │
│  │                                                         │ │
│  │  • By user ID → Partition by user                      │ │
│  │  • By time → Time-series database                      │ │
│  │  • By location → Geospatial indexing                   │ │
│  │  • Full-text search → Elasticsearch                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Non-Functional Requirements

### Scale Questions

```
┌─────────────────────────────────────────────────────────────┐
│                    Scale Questions                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User Scale:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "How many users should this support?"                 │ │
│  │                                                         │ │
│  │  If interviewer says "assume large scale":             │ │
│  │  • Total users: 500M-1B                                │ │
│  │  • DAU: 100M-200M (20% of total)                       │ │
│  │  • Concurrent: 10M (10% of DAU)                        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Traffic Pattern:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Is this read-heavy or write-heavy?"                  │ │
│  │                                                         │ │
│  │  Read-heavy (100:1 ratio):                             │ │
│  │  • Twitter, Instagram (view >> post)                   │ │
│  │  • Strategy: Caching, read replicas                    │ │
│  │                                                         │ │
│  │  Write-heavy (1:1 or more writes):                     │ │
│  │  • IoT systems, logging                                │ │
│  │  • Strategy: Write-optimized DBs, queues              │ │
│  │                                                         │ │
│  │  Mixed:                                                 │ │
│  │  • Chat applications                                   │ │
│  │  • Strategy: Separate read/write paths                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Growth:                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Should we design for 10x growth?"                    │ │
│  │                                                         │ │
│  │  Usually yes - design for next 3-5 years              │ │
│  │  Current: 100K users                                   │ │
│  │  Target: 1M users (10x)                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Performance Questions

```
┌─────────────────────────────────────────────────────────────┐
│                  Performance Questions                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Latency:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "What latency is acceptable?"                         │ │
│  │                                                         │ │
│  │  Real-time (< 100ms):                                  │ │
│  │  • Chat messages, gaming                               │ │
│  │  • Strategy: In-memory, websockets                     │ │
│  │                                                         │ │
│  │  Interactive (< 500ms):                                │ │
│  │  • Web pages, API calls                                │ │
│  │  • Strategy: Caching, CDN                              │ │
│  │                                                         │ │
│  │  Background (seconds-minutes):                         │ │
│  │  • Reports, batch processing                           │ │
│  │  • Strategy: Async processing, queues                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Throughput:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "What throughput do we need to handle?"               │ │
│  │                                                         │ │
│  │  Typical ranges:                                        │ │
│  │  • Small: 100-1K QPS                                   │ │
│  │  • Medium: 1K-100K QPS                                 │ │
│  │  • Large: 100K-1M QPS                                  │ │
│  │  • Massive: 1M+ QPS                                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Reliability Questions

```
┌─────────────────────────────────────────────────────────────┐
│                  Reliability Questions                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Availability:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "What availability do we need?"                       │ │
│  │                                                         │ │
│  │  Availability  | Downtime/Year | Use Case              │ │
│  │  ─────────────────────────────────────────────────────  │ │
│  │  99%          | 3.65 days     | Internal tools        │ │
│  │  99.9%        | 8.76 hours    | Business apps         │ │
│  │  99.99%       | 52.56 min     | E-commerce            │ │
│  │  99.999%      | 5.26 min      | Financial systems     │ │
│  │                                                         │ │
│  │  Higher availability = More complexity + cost          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Durability:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Can we tolerate data loss?"                          │ │
│  │                                                         │ │
│  │  No data loss acceptable:                              │ │
│  │  • Financial transactions, user data                   │ │
│  │  • Strategy: Sync replication, backups                 │ │
│  │                                                         │ │
│  │  Some loss acceptable:                                  │ │
│  │  • Analytics, logs, caches                             │ │
│  │  • Strategy: Async replication, best effort            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Consistency:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  "Strong or eventual consistency?"                     │ │
│  │                                                         │ │
│  │  Strong consistency:                                    │ │
│  │  • Banking, inventory                                  │ │
│  │  • Trade-off: Higher latency                           │ │
│  │                                                         │ │
│  │  Eventual consistency:                                  │ │
│  │  • Social feeds, likes                                 │ │
│  │  • Trade-off: May show stale data                      │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Question Templates by System Type

```
┌─────────────────────────────────────────────────────────────┐
│              System-Specific Questions                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Social Media (Twitter, Instagram):                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  □ What content types? (text, images, videos)          │ │
│  │  □ Character/size limits?                              │ │
│  │  □ How is the feed generated? (chronological, ranked)  │ │
│  │  □ Follow asymmetric or symmetric?                     │ │
│  │  □ How to handle celebrities? (millions of followers)  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Messaging (WhatsApp, Slack):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  □ 1:1 only or groups too?                             │ │
│  │  □ Max group size?                                     │ │
│  │  □ Need read receipts?                                 │ │
│  │  □ Message persistence? (forever vs time-limited)      │ │
│  │  □ End-to-end encryption needed?                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  E-commerce (Amazon):                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  □ Product catalog size?                               │ │
│  │  □ Inventory management real-time?                     │ │
│  │  □ Payment processing in scope?                        │ │
│  │  □ Reviews and ratings?                                │ │
│  │  □ Recommendation system?                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Streaming (Netflix, YouTube):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  □ Live or on-demand?                                  │ │
│  │  □ Video quality levels?                               │ │
│  │  □ Average video duration?                             │ │
│  │  □ User-uploaded or curated?                           │ │
│  │  □ Download for offline?                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Location-based (Uber, Yelp):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  □ Real-time location tracking?                        │ │
│  │  □ Geographic coverage? (global vs regional)           │ │
│  │  □ Location update frequency?                          │ │
│  │  □ Matching algorithm constraints?                     │ │
│  │  □ ETA accuracy requirements?                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Documenting Requirements

```
┌─────────────────────────────────────────────────────────────┐
│              Requirements Documentation                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Write it down! (on whiteboard or paper)                   │
│                                                              │
│  Example format:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  FUNCTIONAL REQUIREMENTS:                               │ │
│  │  ─────────────────────────                              │ │
│  │  • Post tweets (280 chars, images)                     │ │
│  │  • View timeline (home + user)                         │ │
│  │  • Follow/unfollow                                     │ │
│  │  • Search tweets                                       │ │
│  │                                                         │ │
│  │  NON-FUNCTIONAL REQUIREMENTS:                           │ │
│  │  ─────────────────────────────                          │ │
│  │  • 200M DAU                                            │ │
│  │  • 500M tweets/day                                     │ │
│  │  • Read:Write = 100:1                                  │ │
│  │  • Timeline latency < 200ms                            │ │
│  │  • 99.9% availability                                  │ │
│  │  • Eventual consistency OK                             │ │
│  │                                                         │ │
│  │  OUT OF SCOPE:                                          │ │
│  │  ───────────────                                        │ │
│  │  • DMs, trending, ads                                  │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Benefits of writing it down:                              │
│  • Shows organization                                      │
│  • Reference during design                                 │
│  • Interviewer can correct misunderstandings early        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Anti-Patterns

```
┌─────────────────────────────────────────────────────────────┐
│               Requirements Anti-Patterns                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ❌ Assuming without asking:                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bad: "I'll assume we need 1M users"                   │ │
│  │  Good: "How many users should we design for?"          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ❌ Asking too many questions:                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bad: 15 minutes of questions                          │ │
│  │  Good: 3-5 minutes of focused questions                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ❌ Asking obvious questions:                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bad: "Should users be able to log in?"                │ │
│  │  Good: "Should we support social login or just email?" │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ❌ Not documenting answers:                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bad: Keep everything in head                          │ │
│  │  Good: Write key requirements on board                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ❌ Scope creep:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Bad: "We should also add payments, reviews, and..."   │ │
│  │  Good: "These are P0, let's mention P1 at the end"     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Back to Module](./README.md) | [Next: Estimation Techniques →](./03-estimation-techniques.md)
