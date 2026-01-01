# Intermediate System Designs

These designs cover more complex systems with higher scale requirements and sophisticated features.

## Designs in This Section

| # | Design | Key Concepts |
|---|--------|--------------|
| 1 | [Twitter/X](./01-twitter/README.md) | Fan-out, Timeline, Snowflake IDs |
| 2 | [Instagram](./02-instagram/README.md) | Image processing, CDN, Feed ranking |
| 3 | [WhatsApp](./03-whatsapp/README.md) | Real-time messaging, WebSocket, E2EE |
| 4 | [Notification System](./04-notification-system/README.md) | Push/SMS/Email, Priority queues, Templates |
| 5 | [News Feed](./05-news-feed/README.md) | Ranking algorithms, Personalization |
| 6 | [Typeahead](./06-typeahead/README.md) | Trie, Prefix matching, Fuzzy search |
| 7 | [Web Crawler](./07-web-crawler/README.md) | URL frontier, Politeness, Deduplication |

## Common Patterns at This Level

### Fan-Out Strategies
- **Push (Fan-out on write)**: Pre-compute for fast reads
- **Pull (Fan-out on read)**: Compute on demand
- **Hybrid**: Push for regular users, pull for celebrities

### Real-Time Communication
- WebSocket for bidirectional
- Server-Sent Events for one-way
- Long polling as fallback

### Content Delivery
- CDN for static assets
- Edge caching for global latency
- Multi-resolution media storage

### Ranking & Personalization
- ML-based scoring
- Feature extraction
- A/B testing frameworks

## Prerequisites

Before attempting these designs, ensure you understand:
- [Beginner Designs](../beginner/README.md)
- [Building Blocks](../../03-building-blocks/README.md)
- [Design Patterns](../../04-design-patterns/README.md)

---

[Back to System Designs](../README.md) | Next: [Advanced Designs](../advanced/README.md)
