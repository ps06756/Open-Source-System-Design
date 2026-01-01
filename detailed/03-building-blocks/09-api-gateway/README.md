# API Gateway

An API Gateway is a single entry point for all client requests, handling cross-cutting concerns like authentication, rate limiting, and routing.

## Table of Contents
1. [What is an API Gateway?](#what-is-an-api-gateway)
2. [Core Functions](#core-functions)
3. [Popular Solutions](#popular-solutions)
4. [Interview Questions](#interview-questions)

---

## What is an API Gateway?

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Without Gateway:                                            │
│  ┌────────┐    ┌────────┐  ┌────────┐  ┌────────┐          │
│  │ Mobile │───▶│ User   │  │ Order  │  │Payment │          │
│  │  App   │───▶│Service │  │Service │  │Service │          │
│  └────────┘    └────────┘  └────────┘  └────────┘          │
│  Client must know all service locations!                    │
│                                                              │
│  With Gateway:                                               │
│  ┌────────┐    ┌─────────────┐    ┌────────────────┐       │
│  │ Mobile │    │             │───▶│  User Service  │       │
│  │  App   │───▶│ API Gateway │───▶│  Order Service │       │
│  │  Web   │    │             │───▶│ Payment Service│       │
│  └────────┘    └─────────────┘    └────────────────┘       │
│  Single entry point, gateway handles routing!              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Functions

```
┌─────────────────────────────────────────────────────────────┐
│                  API Gateway Functions                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Authentication & Authorization                          │
│     Validate tokens, check permissions                      │
│     Reject unauthorized requests early                      │
│                                                              │
│  2. Rate Limiting                                            │
│     Protect services from abuse                             │
│     Per user, per IP, per API key                          │
│                                                              │
│  3. Request Routing                                          │
│     /users/* → User Service                                 │
│     /orders/* → Order Service                               │
│                                                              │
│  4. Load Balancing                                           │
│     Distribute across service instances                     │
│                                                              │
│  5. Request/Response Transformation                         │
│     Protocol translation (REST ↔ gRPC)                     │
│     Response aggregation from multiple services            │
│                                                              │
│  6. Caching                                                  │
│     Cache common responses                                  │
│                                                              │
│  7. Monitoring & Logging                                     │
│     Centralized metrics and logs                           │
│                                                              │
│  8. Circuit Breaking                                         │
│     Stop calling failing services                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Popular Solutions

| Solution | Best For |
|----------|----------|
| **Kong** | Open source, plugins ecosystem |
| **AWS API Gateway** | Serverless, AWS integration |
| **Apigee** | Enterprise, analytics |
| **Traefik** | Cloud-native, auto-discovery |
| **Nginx** | Performance, simplicity |

---

## Interview Questions

### Basic
1. What is an API Gateway and why is it needed?

### Intermediate
2. How would you implement authentication in an API Gateway?

### Advanced
3. Design an API Gateway for a microservices architecture.

---

[← Previous: Search Engines](../08-search-engines/README.md) | [Next: Rate Limiter →](../10-rate-limiter/README.md)
