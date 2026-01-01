# Module 3: Building Blocks

These are the fundamental components that make up every large-scale system. Understanding when and how to use each building block is essential for system design.

## Why This Matters

Every system design interview involves combining these building blocks:
- "How do you handle millions of requests?" → Load Balancers
- "How do you store user sessions?" → Caching (Redis)
- "How do you handle async processing?" → Message Queues
- "How do you store images/videos?" → Blob Storage + CDN

## Chapters

| Chapter | Component | What You'll Learn |
|---------|-----------|-------------------|
| 1 | [Load Balancers](01-load-balancers/README.md) | L4 vs L7, algorithms, health checks |
| 2 | [Reverse Proxy](02-reverse-proxy/README.md) | Nginx, HAProxy, use cases |
| 3 | [CDN](03-cdn/README.md) | Edge caching, push vs pull, invalidation |
| 4 | [Databases](04-databases/README.md) | SQL vs NoSQL, indexing, selection criteria |
| 5 | [Caching](05-caching/README.md) | Redis, Memcached, cache patterns |
| 6 | [Message Queues](06-message-queues/README.md) | Kafka, RabbitMQ, pub/sub patterns |
| 7 | [Blob Storage](07-blob-storage/README.md) | S3, object storage design |
| 8 | [Search Engines](08-search-engines/README.md) | Elasticsearch, inverted index |
| 9 | [API Gateway](09-api-gateway/README.md) | Authentication, routing, rate limiting |
| 10 | [Rate Limiter](10-rate-limiter/README.md) | Algorithms, distributed limiting |

## Component Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│                When to Use What                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traffic Management:                                         │
│  └── Load Balancer → Distribute across servers              │
│  └── API Gateway → Auth, rate limit, routing                │
│  └── Rate Limiter → Protect from abuse                      │
│                                                              │
│  Performance:                                                │
│  └── CDN → Static content, global users                     │
│  └── Cache → Hot data, reduce DB load                       │
│  └── Reverse Proxy → SSL termination, compression           │
│                                                              │
│  Data Storage:                                               │
│  └── SQL DB → Transactions, complex queries                 │
│  └── NoSQL DB → Scale, flexible schema                      │
│  └── Blob Storage → Large files (images, videos)            │
│  └── Search Engine → Full-text search                       │
│                                                              │
│  Async Processing:                                           │
│  └── Message Queue → Decouple services, async tasks         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Learning Objectives

By the end of this module, you will be able to:

1. Choose the right database for different use cases
2. Design effective caching strategies
3. Implement message-based architectures
4. Understand CDN and edge computing patterns
5. Design rate limiting systems

## Time Investment

- **First-time learning**: 10-15 hours
- **Review**: 2-3 hours

---

[← Previous Module: Core Concepts](../02-core-concepts/README.md) | [Next: Load Balancers →](01-load-balancers/README.md)

[Back to Course](../README.md)
