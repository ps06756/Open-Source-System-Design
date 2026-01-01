# Module 1: Fundamentals

Before diving into distributed systems and system design, you need a solid understanding of how computers communicate over networks. This module covers the essential building blocks that every system design relies on.

## Why This Matters

In system design interviews, interviewers expect you to understand:
- How data travels from a user's browser to your servers
- Why certain architectural decisions affect latency
- The trade-offs between different communication protocols
- How operating systems handle concurrent requests

## Chapters

| Chapter | Topic | What You'll Learn |
|---------|-------|-------------------|
| 1 | [How the Internet Works](01-how-internet-works/README.md) | DNS resolution, TCP/IP stack, HTTP/HTTPS, TLS handshakes |
| 2 | [Client-Server Architecture](02-client-server/README.md) | Request-response patterns, REST, GraphQL, gRPC comparison |
| 3 | [Networking Basics](03-networking/README.md) | Sockets, WebSockets, long polling, Server-Sent Events |
| 4 | [Operating System Concepts](04-os-concepts/README.md) | Processes, threads, I/O models, memory management |

## Prerequisites

- Basic programming knowledge in any language
- Familiarity with using web applications
- No prior networking knowledge required

## Learning Objectives

By the end of this module, you will be able to:

1. Explain the complete journey of a web request from browser to server
2. Choose the right API style (REST, GraphQL, gRPC) for different scenarios
3. Understand when to use WebSockets vs. polling vs. SSE
4. Explain how operating systems handle thousands of concurrent connections

## Time Investment

- **First-time learning**: 4-6 hours
- **Review**: 30-45 minutes

---

[Next: How the Internet Works →](01-how-internet-works/README.md)

[Back to Course](../README.md)
