# Module 5: System Designs

Real-world system design case studies organized by complexity. Each design follows the interview framework and includes architecture diagrams, component breakdowns, and trade-off discussions.

## Difficulty Levels

### Beginner
Foundation designs with clear scope. Good for learning the design process.

| System | Key Concepts |
|--------|--------------|
| [URL Shortener](./beginner/url-shortener.md) | Hashing, Base62, read-heavy workload |
| [Pastebin](./beginner/pastebin.md) | Blob storage, key generation |
| [Rate Limiter](./beginner/rate-limiter.md) | Token bucket, sliding window, distributed |

### Intermediate
Production systems with multiple components and real-world constraints.

| System | Key Concepts |
|--------|--------------|
| [Twitter](./intermediate/twitter.md) | Fan-out, timeline, celebrity problem |
| [Instagram](./intermediate/instagram.md) | Feed ranking, image storage, CDN |
| [WhatsApp](./intermediate/whatsapp.md) | Real-time messaging, presence, E2E encryption |
| [YouTube](./intermediate/youtube.md) | Video processing, streaming, recommendations |
| [Uber](./intermediate/uber.md) | Location, matching, ETA calculation |
| [Dropbox](./intermediate/dropbox.md) | File sync, chunking, deduplication |
| [Netflix](./intermediate/netflix.md) | Streaming, adaptive bitrate, CDN |

### Advanced
Complex systems requiring deep trade-off analysis and multiple subsystems.

| System | Key Concepts |
|--------|--------------|
| [Google Search](./advanced/google-search.md) | Web crawling, indexing, ranking |
| [Distributed Cache](./advanced/distributed-cache.md) | Consistency, partitioning, replication |
| [Payment System](./advanced/payment-system.md) | ACID, idempotency, reconciliation |
| [Ad Platform](./advanced/ad-platform.md) | Real-time bidding, targeting, analytics |
| [Stock Exchange](./advanced/stock-exchange.md) | Order matching, low latency, FIFO |
| [Gaming Platform](./advanced/gaming-platform.md) | Real-time sync, matchmaking, leaderboards |

---

## Design Framework

Use this framework for every system design:

```
┌─────────────────────────────────────────────────────────────┐
│                  System Design Framework                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Requirements (5 min)                                    │
│     • Functional: What does the system do?                 │
│     • Non-functional: Scale, latency, availability         │
│     • Constraints: Budget, existing systems                │
│                                                              │
│  2. Estimation (5 min)                                      │
│     • Users, requests, storage                             │
│     • Back-of-envelope calculations                        │
│                                                              │
│  3. High-Level Design (10 min)                             │
│     • Core components and data flow                        │
│     • API design                                            │
│                                                              │
│  4. Deep Dive (15 min)                                      │
│     • Data model and storage                               │
│     • Key algorithms                                        │
│     • Scaling strategies                                    │
│                                                              │
│  5. Trade-offs & Extensions (5 min)                        │
│     • Design decisions and alternatives                    │
│     • Bottlenecks and solutions                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Previous Module: Design Patterns](../04-design-patterns/README.md) | [Next Module: Interview Strategy →](../06-interview-strategy/README.md)
