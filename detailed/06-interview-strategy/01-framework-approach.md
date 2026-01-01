# Framework & Approach

A structured methodology for tackling any system design interview.

---

## The REDS Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    REDS Framework                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  R - Requirements                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Clarify functional requirements                     │ │
│  │  • Define non-functional requirements                  │ │
│  │  • Establish scope and constraints                     │ │
│  │  • Identify users and use cases                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  E - Estimation                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Calculate scale (users, requests, data)             │ │
│  │  • Estimate storage needs                              │ │
│  │  • Determine bandwidth requirements                    │ │
│  │  • Identify bottlenecks early                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  D - Design                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • High-level architecture                             │ │
│  │  • Core components and APIs                            │ │
│  │  • Data model and storage                              │ │
│  │  • Scale and optimize                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  S - Summary                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Recap the design                                    │ │
│  │  • Discuss trade-offs made                             │ │
│  │  • Mention future improvements                         │ │
│  │  • Address monitoring/alerting                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Requirements (5 minutes)

### Functional Requirements

```
┌─────────────────────────────────────────────────────────────┐
│              Gathering Functional Requirements              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Questions to Ask:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Core Features:                                         │ │
│  │  • "What are the main features we need to support?"    │ │
│  │  • "Who are the primary users?"                        │ │
│  │  • "What actions can users perform?"                   │ │
│  │                                                         │ │
│  │  Scope Boundaries:                                      │ │
│  │  • "Should we focus on read or write operations?"      │ │
│  │  • "Is this mobile, web, or both?"                     │ │
│  │  • "What features are out of scope?"                   │ │
│  │                                                         │ │
│  │  Data:                                                  │ │
│  │  • "What data do we need to store?"                    │ │
│  │  • "How long should we retain data?"                   │ │
│  │  • "What are the access patterns?"                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Example (Twitter):                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Functional Requirements:                               │ │
│  │  ✓ Post tweets (text, images)                          │ │
│  │  ✓ Follow/unfollow users                               │ │
│  │  ✓ View home timeline                                  │ │
│  │  ✓ View user profile                                   │ │
│  │                                                         │ │
│  │  Out of Scope (for now):                               │ │
│  │  ✗ Direct messages                                     │ │
│  │  ✗ Trending topics                                     │ │
│  │  ✗ Ads and monetization                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Non-Functional Requirements

```
┌─────────────────────────────────────────────────────────────┐
│            Gathering Non-Functional Requirements            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Key Categories:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Scale:                                                 │ │
│  │  • "How many users? Daily active users?"               │ │
│  │  • "How many requests per second?"                     │ │
│  │  • "Read-heavy or write-heavy?"                        │ │
│  │                                                         │ │
│  │  Performance:                                           │ │
│  │  • "What latency is acceptable?"                       │ │
│  │  • "Any real-time requirements?"                       │ │
│  │                                                         │ │
│  │  Reliability:                                           │ │
│  │  • "What availability do we need? (99.9%?)"            │ │
│  │  • "Can we tolerate data loss?"                        │ │
│  │                                                         │ │
│  │  Consistency:                                           │ │
│  │  • "Strong or eventual consistency?"                   │ │
│  │  • "What happens during network partitions?"           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  The CAP Trade-off:                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │            Consistency                                  │ │
│  │                △                                        │ │
│  │               ╱ ╲                                       │ │
│  │              ╱   ╲                                      │ │
│  │             ╱     ╲                                     │ │
│  │            ╱   P   ╲    P = Partition Tolerance         │ │
│  │           ╱─────────╲   (always required)               │ │
│  │          ╱           ╲                                  │ │
│  │  Availability ──────── Partition                        │ │
│  │                        Tolerance                        │ │
│  │                                                         │ │
│  │  Choose: CP (consistent) or AP (available)             │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 2: Estimation (3-5 minutes)

```
┌─────────────────────────────────────────────────────────────┐
│                  Back-of-Envelope Math                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traffic Estimation:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given: 100M DAU, each user makes 10 requests/day      │ │
│  │                                                         │ │
│  │  Daily requests = 100M × 10 = 1B requests/day          │ │
│  │                                                         │ │
│  │  QPS (avg) = 1B / 86,400 ≈ 12K QPS                     │ │
│  │                                                         │ │
│  │  QPS (peak) = 12K × 3 ≈ 36K QPS                        │ │
│  │  (assume 3x peak factor)                                │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Storage Estimation:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given: 10M new items/day, each 1KB                    │ │
│  │                                                         │ │
│  │  Daily storage = 10M × 1KB = 10GB/day                  │ │
│  │                                                         │ │
│  │  Yearly storage = 10GB × 365 ≈ 3.6TB/year              │ │
│  │                                                         │ │
│  │  5-year storage = 3.6TB × 5 = 18TB                     │ │
│  │  (plus replication factor)                              │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Bandwidth Estimation:                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Given: 36K QPS peak, 100KB average response           │ │
│  │                                                         │ │
│  │  Bandwidth = 36K × 100KB = 3.6GB/s outbound            │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 3: Design (25-30 minutes)

### 3a. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 High-Level Design Phase                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Start with the big picture:                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Draw main components as boxes                      │ │
│  │     • Clients                                          │ │
│  │     • Load balancers                                   │ │
│  │     • Application servers                              │ │
│  │     • Databases                                        │ │
│  │     • Caches                                           │ │
│  │                                                         │ │
│  │  2. Show data flow with arrows                         │ │
│  │     • Request path                                     │ │
│  │     • Response path                                    │ │
│  │     • Async operations                                 │ │
│  │                                                         │ │
│  │  3. Label everything clearly                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Generic Starting Template:                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  ┌─────────┐     ┌────────┐     ┌──────────────┐      │ │
│  │  │ Clients │────▶│  LB    │────▶│ App Servers  │      │ │
│  │  └─────────┘     └────────┘     └──────┬───────┘      │ │
│  │                                        │               │ │
│  │                              ┌─────────┼─────────┐    │ │
│  │                              ▼         ▼         ▼    │ │
│  │                          ┌──────┐ ┌───────┐ ┌──────┐ │ │
│  │                          │Cache │ │  DB   │ │Queue │ │ │
│  │                          └──────┘ └───────┘ └──────┘ │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Get interviewer buy-in before diving deeper!              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3b. Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                  Deep Dive Areas                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  API Design:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  REST Example:                                          │ │
│  │  POST /tweets                                           │ │
│  │  { "user_id": "123", "content": "Hello world" }        │ │
│  │                                                         │ │
│  │  GET /timeline?user_id=123&page=1&limit=20             │ │
│  │  Returns: { "tweets": [...], "next_cursor": "abc" }    │ │
│  │                                                         │ │
│  │  Consider:                                              │ │
│  │  • Pagination (cursor vs offset)                       │ │
│  │  • Rate limiting                                        │ │
│  │  • Authentication                                       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Data Model:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Define key entities:                                   │ │
│  │                                                         │ │
│  │  User {                                                 │ │
│  │    id: UUID,                                           │ │
│  │    username: string,                                   │ │
│  │    email: string,                                      │ │
│  │    created_at: timestamp                               │ │
│  │  }                                                      │ │
│  │                                                         │ │
│  │  Tweet {                                                │ │
│  │    id: UUID,                                           │ │
│  │    user_id: UUID,                                      │ │
│  │    content: string,                                    │ │
│  │    created_at: timestamp                               │ │
│  │  }                                                      │ │
│  │                                                         │ │
│  │  Consider:                                              │ │
│  │  • Primary keys and indexes                            │ │
│  │  • Relationships                                        │ │
│  │  • Denormalization needs                               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Database Choice:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  SQL vs NoSQL Decision Matrix:                         │ │
│  │                                                         │ │
│  │  Choose SQL when:                                       │ │
│  │  • Complex queries needed                              │ │
│  │  • ACID transactions required                          │ │
│  │  • Data is relational                                  │ │
│  │                                                         │ │
│  │  Choose NoSQL when:                                     │ │
│  │  • High write throughput                               │ │
│  │  • Flexible schema                                     │ │
│  │  • Horizontal scaling priority                         │ │
│  │                                                         │ │
│  │  Always justify your choice!                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3c. Scaling

```
┌─────────────────────────────────────────────────────────────┐
│                   Scaling Strategies                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Horizontal Scaling:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  App Servers:                                           │ │
│  │  • Add more servers behind load balancer               │ │
│  │  • Keep servers stateless                              │ │
│  │  • Use auto-scaling                                    │ │
│  │                                                         │ │
│  │  Databases:                                             │ │
│  │  • Read replicas for read scaling                      │ │
│  │  • Sharding for write scaling                          │ │
│  │  • Partition key selection is critical                 │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Caching:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  What to cache:                                         │ │
│  │  • Frequently read data                                │ │
│  │  • Expensive computations                              │ │
│  │  • Session data                                        │ │
│  │                                                         │ │
│  │  Cache strategies:                                      │ │
│  │  • Cache-aside (lazy loading)                          │ │
│  │  • Write-through                                        │ │
│  │  • Write-behind                                         │ │
│  │                                                         │ │
│  │  Invalidation:                                          │ │
│  │  • TTL-based                                            │ │
│  │  • Event-based                                          │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Async Processing:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Use message queues for:                               │ │
│  │  • Decoupling services                                 │ │
│  │  • Handling traffic spikes                             │ │
│  │  • Background processing                               │ │
│  │                                                         │ │
│  │  Examples:                                              │ │
│  │  • Sending notifications                               │ │
│  │  • Processing uploads                                  │ │
│  │  • Analytics events                                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 4: Summary (2-3 minutes)

```
┌─────────────────────────────────────────────────────────────┐
│                     Wrap-Up Checklist                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✓ Recap:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  "To summarize, we designed a system that..."          │ │
│  │  • Handles X requests per second                       │ │
│  │  • Stores Y data with Z retention                      │ │
│  │  • Achieves latency under N milliseconds               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ✓ Trade-offs:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  "Key trade-offs we made..."                           │ │
│  │  • Chose eventual consistency for availability         │ │
│  │  • Used NoSQL for write throughput over joins          │ │
│  │  • Prioritized read performance with caching           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ✓ Future Improvements:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  "If we had more time, we could add..."                │ │
│  │  • ML-based recommendations                            │ │
│  │  • Multi-region deployment                             │ │
│  │  • Advanced analytics                                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ✓ Monitoring:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  "For production, we'd monitor..."                     │ │
│  │  • Request latency (p50, p99)                          │ │
│  │  • Error rates                                         │ │
│  │  • Database connection pools                           │ │
│  │  • Cache hit rates                                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Framework Cheat Sheet

```
┌─────────────────────────────────────────────────────────────┐
│                Quick Reference Card                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REQUIREMENTS (5 min)                                       │
│  □ Who are the users?                                       │
│  □ What are core features?                                  │
│  □ What's out of scope?                                     │
│  □ How many users? QPS?                                     │
│  □ Latency requirements?                                    │
│  □ Availability requirements?                               │
│                                                              │
│  ESTIMATION (3 min)                                         │
│  □ Daily/monthly active users                               │
│  □ Requests per second (avg, peak)                          │
│  □ Storage needs (daily, yearly)                            │
│  □ Bandwidth requirements                                   │
│                                                              │
│  DESIGN (25 min)                                            │
│  □ High-level architecture diagram                          │
│  □ API design (key endpoints)                               │
│  □ Data model (key entities)                                │
│  □ Database choice (SQL vs NoSQL)                           │
│  □ Caching strategy                                         │
│  □ Scaling approach                                         │
│                                                              │
│  DEEP DIVE (10 min)                                         │
│  □ Address interviewer questions                            │
│  □ Handle edge cases                                        │
│  □ Discuss failures and recovery                            │
│                                                              │
│  SUMMARY (2 min)                                            │
│  □ Recap key points                                         │
│  □ State trade-offs                                         │
│  □ Mention improvements                                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Back to Module](./README.md) | [Next: Requirements Gathering →](./02-requirements-gathering.md)
