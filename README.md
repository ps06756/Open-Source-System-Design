# Open Source System Design Course

A comprehensive, free system design course to help you ace Big Tech interviews at companies like Google, Meta, Amazon, Apple, Netflix, and Microsoft.

## Why This Course?

System design interviews are a critical part of senior engineering interviews. This course provides:

- **Structured learning path** from fundamentals to advanced designs
- **Real interview questions** with detailed solutions
- **Trade-off discussions** that interviewers look for
- **Back-of-envelope calculations** with practical examples
- **Open source** - contribute and learn together

## Course Structure

### Module 1: Fundamentals
Build a strong foundation with networking, APIs, and core internet concepts.

| Topic | Description |
|-------|-------------|
| [How the Internet Works](01-fundamentals/01-how-internet-works/README.md) | DNS, TCP/IP, HTTP/HTTPS, TLS |
| [Client-Server Architecture](01-fundamentals/02-client-server/README.md) | Request-response, APIs (REST, GraphQL, gRPC) |
| [Networking Basics](01-fundamentals/03-networking/README.md) | Sockets, WebSockets, long polling, SSE |
| [Operating System Concepts](01-fundamentals/04-os-concepts/README.md) | Processes, threads, memory, I/O |

### Module 2: Core Concepts
Master the essential distributed systems concepts that appear in every interview.

| Topic | Description |
|-------|-------------|
| [Scalability](02-core-concepts/01-scalability/README.md) | Horizontal vs vertical scaling |
| [Availability & Reliability](02-core-concepts/02-availability/README.md) | SLAs, SLOs, fault tolerance |
| [Consistency Models](02-core-concepts/03-consistency/README.md) | Strong, eventual, causal consistency |
| [CAP Theorem & PACELC](02-core-concepts/04-cap-theorem/README.md) | Trade-offs in distributed systems |
| [Latency vs Throughput](02-core-concepts/05-latency-throughput/README.md) | Performance metrics |
| [Data Partitioning](02-core-concepts/06-partitioning/README.md) | Sharding, consistent hashing |
| [Replication](02-core-concepts/07-replication/README.md) | Leader-follower, multi-leader |
| [Consensus](02-core-concepts/08-consensus/README.md) | Paxos, Raft, leader election |
| [Back-of-envelope Calculations](02-core-concepts/09-calculations/README.md) | QPS, storage, bandwidth |

### Module 3: Building Blocks
Learn the components that make up every large-scale system.

| Component | Description |
|-----------|-------------|
| [Load Balancers](03-building-blocks/01-load-balancers/README.md) | L4 vs L7, algorithms |
| [Reverse Proxy](03-building-blocks/02-reverse-proxy/README.md) | Nginx, HAProxy |
| [CDN](03-building-blocks/03-cdn/README.md) | Edge caching, push vs pull |
| [Databases](03-building-blocks/04-databases/README.md) | SQL vs NoSQL, indexing |
| [Caching](03-building-blocks/05-caching/README.md) | Redis, cache patterns |
| [Message Queues](03-building-blocks/06-message-queues/README.md) | Kafka, RabbitMQ |
| [Blob Storage](03-building-blocks/07-blob-storage/README.md) | S3, object storage |
| [Search Engines](03-building-blocks/08-search-engines/README.md) | Elasticsearch, inverted index |
| [API Gateway](03-building-blocks/09-api-gateway/README.md) | Auth, rate limiting |
| [Rate Limiter](03-building-blocks/10-rate-limiter/README.md) | Token bucket, sliding window |

### Module 4: Design Patterns
Understand architectural patterns used in production systems.

| Pattern | Description |
|---------|-------------|
| [Microservices vs Monolith](04-design-patterns/01-microservices-monolith/README.md) | When to use each |
| [Event-Driven Architecture](04-design-patterns/02-event-driven/README.md) | Async, event sourcing |
| [CQRS](04-design-patterns/03-cqrs/README.md) | Separating reads and writes |
| [Saga Pattern](04-design-patterns/04-saga/README.md) | Distributed transactions |
| [Circuit Breaker](04-design-patterns/05-circuit-breaker/README.md) | Fault tolerance |
| [Sidecar & Ambassador](04-design-patterns/06-sidecar-ambassador/README.md) | Service mesh patterns |

### Module 5: System Designs
Practice with real interview questions, organized by difficulty.

#### Beginner
| System | Key Concepts |
|--------|--------------|
| [URL Shortener](05-system-designs/beginner/01-url-shortener/README.md) | Hashing, encoding, databases |
| [Pastebin](05-system-designs/beginner/02-pastebin/README.md) | Object storage, expiration |
| [Rate Limiter](05-system-designs/beginner/03-rate-limiter/README.md) | Algorithms, distributed limiting |
| [Key-Value Store](05-system-designs/beginner/04-key-value-store/README.md) | Storage engines, replication |

#### Intermediate
| System | Key Concepts |
|--------|--------------|
| [Twitter / X](05-system-designs/intermediate/01-twitter/README.md) | Feed generation, fan-out |
| [Instagram](05-system-designs/intermediate/02-instagram/README.md) | Image storage, CDN |
| [WhatsApp](05-system-designs/intermediate/03-whatsapp/README.md) | Real-time messaging, presence |
| [Notification System](05-system-designs/intermediate/04-notifications/README.md) | Push, priority queues |
| [News Feed](05-system-designs/intermediate/05-news-feed/README.md) | Ranking, personalization |
| [Typeahead](05-system-designs/intermediate/06-typeahead/README.md) | Trie, prefix matching |
| [Web Crawler](05-system-designs/intermediate/07-web-crawler/README.md) | Politeness, deduplication |

#### Advanced
| System | Key Concepts |
|--------|--------------|
| [YouTube](05-system-designs/advanced/01-youtube/README.md) | Video processing, streaming |
| [Uber](05-system-designs/advanced/02-uber/README.md) | Location, matching, ETA |
| [Google Maps](05-system-designs/advanced/03-google-maps/README.md) | Graph algorithms, tiles |
| [Dropbox](05-system-designs/advanced/04-dropbox/README.md) | Sync, chunking, dedup |
| [Distributed Cache](05-system-designs/advanced/05-distributed-cache/README.md) | Consistent hashing |
| [Distributed Queue](05-system-designs/advanced/06-distributed-queue/README.md) | Ordering, delivery |
| [Payment System](05-system-designs/advanced/07-payment-system/README.md) | ACID, idempotency |
| [Hotel Booking](05-system-designs/advanced/08-hotel-booking/README.md) | Inventory, double booking |
| [Search Engine](05-system-designs/advanced/09-search-engine/README.md) | Crawling, indexing, ranking |
| [Ad Click Aggregator](05-system-designs/advanced/10-ad-aggregator/README.md) | Stream processing |
| [Task Scheduler](05-system-designs/advanced/11-task-scheduler/README.md) | Cron, distribution |
| [Stock Exchange](05-system-designs/advanced/12-stock-exchange/README.md) | Order matching, latency |

### Module 6: Interview Strategy
Learn how to approach and ace system design interviews.

| Topic | Description |
|-------|-------------|
| [Framework](06-interview-strategy/01-framework/README.md) | Step-by-step approach |
| [Requirements Gathering](06-interview-strategy/02-requirements/README.md) | Clarifying questions |
| [Capacity Estimation](06-interview-strategy/03-capacity/README.md) | Quick math techniques |
| [High-Level Design](06-interview-strategy/04-high-level/README.md) | Starting broad |
| [Deep Dives](06-interview-strategy/05-deep-dives/README.md) | When and how |
| [Trade-offs](06-interview-strategy/06-tradeoffs/README.md) | Articulating decisions |
| [Common Mistakes](06-interview-strategy/07-mistakes/README.md) | What to avoid |

### Resources
| Resource | Description |
|----------|-------------|
| [Cheat Sheets](resources/cheat-sheets/) | Quick reference guides |
| [Diagrams](resources/diagrams/) | Reusable architecture diagrams |
| [Interview Rubric](resources/interview-rubric.md) | How companies evaluate |
| [Reading List](resources/reading-list.md) | Papers, blogs, books |

## How to Use This Course

1. **Beginners**: Start with Module 1 and work through sequentially
2. **Intermediate**: Review Module 2-3, then practice with system designs
3. **Interview prep**: Focus on Module 5-6, use others as reference

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) for details.
