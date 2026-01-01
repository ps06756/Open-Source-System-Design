# System Design - Detailed Learning

Comprehensive learning materials with in-depth explanations, real-world examples, and thorough coverage of each topic.

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
| [Microservices](04-design-patterns/01-microservices/README.md) | When and how to decompose |
| [Event-Driven Architecture](04-design-patterns/02-event-driven/README.md) | Async, event sourcing |
| [CQRS & Event Sourcing](04-design-patterns/03-cqrs-event-sourcing/README.md) | Separating reads and writes |
| [Saga Pattern](04-design-patterns/04-saga-pattern/README.md) | Distributed transactions |
| [Circuit Breaker](04-design-patterns/05-circuit-breaker/README.md) | Fault tolerance |
| [Sidecar & Service Mesh](04-design-patterns/06-sidecar-service-mesh/README.md) | Service mesh patterns |

### Module 5: System Designs
Practice with real interview questions, organized by difficulty.

#### Beginner
| System | Key Concepts |
|--------|--------------|
| [URL Shortener](05-system-designs/beginner/url-shortener.md) | Hashing, encoding, databases |
| [Pastebin](05-system-designs/beginner/pastebin.md) | Object storage, expiration |
| [Rate Limiter](05-system-designs/beginner/rate-limiter.md) | Algorithms, distributed limiting |

#### Intermediate
| System | Key Concepts |
|--------|--------------|
| [Twitter / X](05-system-designs/intermediate/twitter.md) | Feed generation, fan-out |
| [Instagram](05-system-designs/intermediate/instagram.md) | Image storage, CDN |
| [WhatsApp](05-system-designs/intermediate/whatsapp.md) | Real-time messaging, presence |
| [YouTube](05-system-designs/intermediate/youtube.md) | Video processing, streaming |
| [Uber](05-system-designs/intermediate/uber.md) | Location, matching, ETA |
| [Dropbox](05-system-designs/intermediate/dropbox.md) | Sync, chunking, dedup |
| [Netflix](05-system-designs/intermediate/netflix.md) | CDN, adaptive streaming |

#### Advanced
| System | Key Concepts |
|--------|--------------|
| [Google Search](05-system-designs/advanced/google-search.md) | Crawling, indexing, ranking |
| [Distributed Cache](05-system-designs/advanced/distributed-cache.md) | Consistent hashing |
| [Payment System](05-system-designs/advanced/payment-system.md) | ACID, idempotency |
| [Ad Platform](05-system-designs/advanced/ad-platform.md) | Real-time bidding, targeting |
| [Stock Exchange](05-system-designs/advanced/stock-exchange.md) | Order matching, latency |
| [Gaming Platform](05-system-designs/advanced/gaming-platform.md) | Real-time sync, matchmaking |

### Module 6: Interview Strategy
Learn how to approach and ace system design interviews.

| Topic | Description |
|-------|-------------|
| [Framework & Approach](06-interview-strategy/01-framework-approach.md) | Step-by-step methodology |
| [Requirements Gathering](06-interview-strategy/02-requirements-gathering.md) | Clarifying questions |
| [Estimation Techniques](06-interview-strategy/03-estimation-techniques.md) | Quick math techniques |
| [Communication Tips](06-interview-strategy/04-communication-tips.md) | Collaborating with interviewer |
| [Common Mistakes](06-interview-strategy/05-common-mistakes.md) | What to avoid |
| [Practice Problems](06-interview-strategy/06-practice-problems.md) | Self-assessment exercises |
| [Mock Interview Guide](06-interview-strategy/07-mock-interview-guide.md) | Simulating real interviews |

---

## How to Use

1. **Complete beginners**: Start with Module 1 and work through sequentially
2. **Some experience**: Review Module 2-3, then practice with system designs
3. **Interview prep**: Focus on Module 5-6, use others as reference

Each topic includes:
- Detailed explanations with real-world context
- Diagrams and visualizations (ASCII art)
- Common interview questions
- Trade-off discussions

---

[Back to Main](../README.md) | [Go to Quick Revision](../revision/README.md)
