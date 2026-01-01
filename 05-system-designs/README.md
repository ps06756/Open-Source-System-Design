# Module 5: System Designs

This module contains real system design interview questions with detailed solutions. Practice these to prepare for big tech interviews.

## Difficulty Levels

### Beginner
Start here if you're new to system design. These problems focus on core concepts.

| System | Key Concepts |
|--------|--------------|
| [URL Shortener](beginner/01-url-shortener/README.md) | Hashing, encoding, databases |
| [Pastebin](beginner/02-pastebin/README.md) | Object storage, expiration |
| [Rate Limiter](beginner/03-rate-limiter/README.md) | Algorithms, distributed limiting |
| [Key-Value Store](beginner/04-key-value-store/README.md) | Storage engines, replication |

### Intermediate
More complex systems with multiple components and trade-offs.

| System | Key Concepts |
|--------|--------------|
| [Twitter / X](intermediate/01-twitter/README.md) | Feed generation, fan-out |
| [Instagram](intermediate/02-instagram/README.md) | Image storage, CDN |
| [WhatsApp](intermediate/03-whatsapp/README.md) | Real-time messaging, presence |
| [Notification System](intermediate/04-notifications/README.md) | Push, priority queues |
| [News Feed](intermediate/05-news-feed/README.md) | Ranking, personalization |
| [Typeahead](intermediate/06-typeahead/README.md) | Trie, prefix matching |
| [Web Crawler](intermediate/07-web-crawler/README.md) | Politeness, deduplication |

### Advanced
Complex, large-scale systems with sophisticated requirements.

| System | Key Concepts |
|--------|--------------|
| [YouTube](advanced/01-youtube/README.md) | Video processing, streaming |
| [Uber](advanced/02-uber/README.md) | Location, matching, ETA |
| [Google Maps](advanced/03-google-maps/README.md) | Graph algorithms, tiles |
| [Dropbox](advanced/04-dropbox/README.md) | Sync, chunking, dedup |
| [Distributed Cache](advanced/05-distributed-cache/README.md) | Consistent hashing |
| [Distributed Queue](advanced/06-distributed-queue/README.md) | Ordering, delivery |
| [Payment System](advanced/07-payment-system/README.md) | ACID, idempotency |
| [Hotel Booking](advanced/08-hotel-booking/README.md) | Inventory, double booking |
| [Search Engine](advanced/09-search-engine/README.md) | Crawling, indexing, ranking |
| [Ad Click Aggregator](advanced/10-ad-aggregator/README.md) | Stream processing |
| [Task Scheduler](advanced/11-task-scheduler/README.md) | Cron, distribution |
| [Stock Exchange](advanced/12-stock-exchange/README.md) | Order matching, latency |

## How to Use This Module

### For Learning

1. **Read the problem** first without looking at the solution
2. **Spend 30-45 minutes** designing your own solution
3. **Compare** with the provided solution
4. **Understand the trade-offs** discussed

### For Interview Prep

1. **Practice explaining** out loud (or with a partner)
2. **Time yourself** - interviews are 45-60 minutes
3. **Start with beginner**, progress to advanced
4. **Focus on your weaknesses** (scale? databases? real-time?)

## Common Interview Framework

For each design, follow this structure:

```
1. Requirements Clarification (5 min)
   - Functional requirements
   - Non-functional requirements
   - Scale estimation

2. High-Level Design (10-15 min)
   - Core components
   - Data flow
   - API design

3. Detailed Design (15-20 min)
   - Database schema
   - Algorithm choices
   - Component deep-dive

4. Trade-offs & Discussion (10 min)
   - Scalability
   - Reliability
   - Alternatives considered
```

## Prerequisites

Complete these modules first:
- [Module 2: Core Concepts](../02-core-concepts/README.md)
- [Module 3: Building Blocks](../03-building-blocks/README.md)
- [Module 4: Design Patterns](../04-design-patterns/README.md)

## Next Steps

After practicing designs, see [Module 6: Interview Strategy](../06-interview-strategy/README.md) for tips on how to present your designs effectively.
