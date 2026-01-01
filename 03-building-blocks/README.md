# Module 3: Building Blocks

These are the infrastructure components that make up large-scale systems. Understanding each component's capabilities and trade-offs is essential for system design.

## Topics

1. [Load Balancers](01-load-balancers/README.md) - L4 vs L7, algorithms
2. [Reverse Proxy](02-reverse-proxy/README.md) - Nginx, HAProxy
3. [CDN](03-cdn/README.md) - Edge caching, push vs pull
4. [Databases](04-databases/README.md) - SQL vs NoSQL, indexing
5. [Caching](05-caching/README.md) - Redis, cache patterns
6. [Message Queues](06-message-queues/README.md) - Kafka, RabbitMQ
7. [Blob Storage](07-blob-storage/README.md) - S3, object storage
8. [Search Engines](08-search-engines/README.md) - Elasticsearch
9. [API Gateway](09-api-gateway/README.md) - Auth, rate limiting
10. [Rate Limiter](10-rate-limiter/README.md) - Algorithms, distributed limiting

## Why These Matter

In system design interviews, you'll combine these building blocks to create complete solutions. You need to understand:
- When to use each component
- How to configure them
- Their limitations and trade-offs
- How they interact with each other

## Component Selection Guide

| Need | Component |
|------|-----------|
| Distribute traffic | Load Balancer |
| Cache static content | CDN |
| Store structured data | Database (SQL/NoSQL) |
| Speed up reads | Cache (Redis) |
| Decouple services | Message Queue |
| Store files | Blob Storage |
| Full-text search | Search Engine |
| Protect APIs | API Gateway + Rate Limiter |

## Prerequisites

Complete [Module 2: Core Concepts](../02-core-concepts/README.md) first.

## Next Steps

After this module, proceed to [Module 4: Design Patterns](../04-design-patterns/README.md).
