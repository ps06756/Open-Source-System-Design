# Microservices Architecture

Microservices is an architectural style where an application is built as a collection of small, independent services that communicate over well-defined APIs.

## Table of Contents
1. [Monolith vs Microservices](#monolith-vs-microservices)
2. [Service Decomposition](#service-decomposition)
3. [Communication Patterns](#communication-patterns)
4. [Data Management](#data-management)
5. [Challenges](#challenges)
6. [Interview Questions](#interview-questions)

---

## Monolith vs Microservices

```
┌─────────────────────────────────────────────────────────────┐
│              Monolith vs Microservices                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Monolith:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   Single Application                    │ │
│  │  ┌──────────┬──────────┬──────────┬──────────────────┐│ │
│  │  │   User   │  Order   │ Payment  │    Inventory     ││ │
│  │  │  Module  │  Module  │  Module  │     Module       ││ │
│  │  └──────────┴──────────┴──────────┴──────────────────┘│ │
│  │                    Shared Database                     │ │
│  └────────────────────────────────────────────────────────┘ │
│  • Single deployment unit                                   │
│  • Shared memory, direct function calls                    │
│  • One technology stack                                     │
│                                                              │
│  Microservices:                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   User   │  │  Order   │  │ Payment  │  │Inventory │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │             │             │          │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐      │
│  │   DB    │  │   DB    │  │   DB    │  │   DB    │      │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
│  • Independent deployment                                   │
│  • Network communication (HTTP, gRPC, events)              │
│  • Polyglot (different languages/databases)                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Comparison

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | All or nothing | Independent |
| **Scaling** | Scale entire app | Scale individual services |
| **Technology** | Single stack | Polyglot |
| **Team Structure** | Feature teams | Service teams |
| **Complexity** | In code | In operations |
| **Latency** | Function calls | Network calls |
| **Data Consistency** | ACID transactions | Eventual consistency |
| **Best For** | Small teams, startups | Large orgs, complex domains |

---

## Service Decomposition

```
┌─────────────────────────────────────────────────────────────┐
│               Service Decomposition                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Strategy 1: Decompose by Business Capability               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  E-commerce Company                                    │ │
│  │  ├── Product Catalog (what we sell)                   │ │
│  │  ├── Inventory (what we have)                         │ │
│  │  ├── Order Management (customer orders)               │ │
│  │  ├── Shipping (delivery)                              │ │
│  │  ├── Customer Service (support)                       │ │
│  │  └── Billing (payments)                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Strategy 2: Decompose by Subdomain (DDD)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Core Domain:     Order, Payment (competitive edge)   │ │
│  │  Supporting:      Inventory, Catalog (necessary)      │ │
│  │  Generic:         Auth, Notifications (commodity)     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Signs of Good Boundaries:                                  │
│  ✓ Low coupling (few dependencies between services)       │
│  ✓ High cohesion (related things stay together)           │
│  ✓ Single responsibility                                   │
│  ✓ Independent data ownership                              │
│  ✓ Aligned with team structure                            │
│                                                              │
│  Signs of Bad Boundaries:                                   │
│  ✗ Frequent cross-service transactions                    │
│  ✗ Chatty communication between services                  │
│  ✗ Shared database tables                                 │
│  ✗ Circular dependencies                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Service Size Guidelines

```
┌─────────────────────────────────────────────────────────────┐
│                   How Big Should a Service Be?              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Too Small (Nanoservices):                                  │
│  • High network overhead                                    │
│  • Operational complexity                                   │
│  • Hard to reason about system                             │
│                                                              │
│  Too Large (Distributed Monolith):                         │
│  • Still tightly coupled                                   │
│  • Can't deploy independently                              │
│  • Same problems as monolith                               │
│                                                              │
│  Right Size:                                                │
│  • "Two-pizza team" can own it                             │
│  • Can be rewritten in 2 weeks                             │
│  • Has clear business purpose                              │
│  • Can be deployed independently                           │
│                                                              │
│  Start with fewer, larger services.                        │
│  Split when there's a clear need.                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Communication Patterns

```
┌─────────────────────────────────────────────────────────────┐
│               Communication Patterns                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Synchronous (Request-Response)                         │
│  ┌──────────┐    HTTP/gRPC    ┌──────────┐                 │
│  │ Service  │───────────────▶│ Service  │                 │
│  │    A     │◀───────────────│    B     │                 │
│  └──────────┘    Response     └──────────┘                 │
│                                                              │
│  ✓ Simple mental model                                     │
│  ✓ Immediate response                                      │
│  ✗ Temporal coupling (both must be up)                    │
│  ✗ Cascading failures                                     │
│                                                              │
│  2. Asynchronous (Events/Messages)                         │
│  ┌──────────┐    Event    ┌─────────┐    ┌──────────┐     │
│  │ Service  │────────────▶│ Message │───▶│ Service  │     │
│  │    A     │             │  Broker │    │    B     │     │
│  └──────────┘             └─────────┘    └──────────┘     │
│                                                              │
│  ✓ Loose coupling                                          │
│  ✓ Better resilience                                       │
│  ✓ Scalability                                             │
│  ✗ Eventual consistency                                   │
│  ✗ Harder to debug                                        │
│                                                              │
│  3. Hybrid Approach (Common in Practice)                   │
│  ┌────────────────────────────────────────────────────────┐│
│  │  Sync for:                                             ││
│  │  • Queries needing immediate response                 ││
│  │  • Simple CRUD operations                             ││
│  │                                                        ││
│  │  Async for:                                            ││
│  │  • Events (OrderPlaced, UserCreated)                  ││
│  │  • Long-running operations                            ││
│  │  • Cross-service data sync                            ││
│  └────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### API Design

```
┌─────────────────────────────────────────────────────────────┐
│                    API Design                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REST                                                        │
│  • Resource-based: /users/{id}/orders                      │
│  • HTTP verbs: GET, POST, PUT, DELETE                      │
│  • Best for: Public APIs, CRUD operations                  │
│                                                              │
│  gRPC                                                        │
│  • Binary protocol (Protocol Buffers)                      │
│  • Strongly typed, code generation                         │
│  • Best for: Internal services, high performance           │
│                                                              │
│  GraphQL                                                     │
│  • Client specifies data needed                            │
│  • Single endpoint                                          │
│  • Best for: Mobile apps, complex queries                  │
│                                                              │
│  Versioning Strategies:                                     │
│  • URL path: /v1/users, /v2/users                          │
│  • Header: Accept: application/vnd.api+json;version=1     │
│  • Query param: /users?version=1                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Management

```
┌─────────────────────────────────────────────────────────────┐
│               Data Management                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Rule: Each service owns its data (Database per Service)   │
│                                                              │
│  ┌───────────┐        ┌───────────┐        ┌───────────┐   │
│  │  Order    │        │  User     │        │  Product  │   │
│  │  Service  │        │  Service  │        │  Service  │   │
│  └─────┬─────┘        └─────┬─────┘        └─────┬─────┘   │
│        │                    │                    │          │
│  ┌─────┴─────┐        ┌─────┴─────┐        ┌─────┴─────┐   │
│  │  Orders   │        │   Users   │        │ Products  │   │
│  │   (SQL)   │        │  (NoSQL)  │        │  (NoSQL)  │   │
│  └───────────┘        └───────────┘        └───────────┘   │
│                                                              │
│  Challenge: Cross-service queries                           │
│                                                              │
│  Anti-Pattern: Direct database access                       │
│  ┌───────────┐                                              │
│  │  Order    │──────▶ Users DB  ✗ DON'T DO THIS!          │
│  │  Service  │                                              │
│  └───────────┘                                              │
│                                                              │
│  Solutions:                                                  │
│  1. API Composition: Query multiple services, aggregate    │
│  2. CQRS: Maintain read-optimized views                    │
│  3. Event Sourcing: Replicate data via events              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Data Consistency Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              Data Consistency Patterns                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Saga Pattern (for distributed transactions):              │
│  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐     │
│  │ Create │───▶│Reserve │───▶│ Charge │───▶│  Ship  │     │
│  │ Order  │    │Inventory│   │Payment │    │ Order  │     │
│  └────────┘    └────────┘    └────────┘    └────────┘     │
│       │             │             │             │          │
│       ▼             ▼             ▼             ▼          │
│  If failure: Execute compensating transactions             │
│  [Cancel]◀──[Release]◀────[Refund]◀────[Cancel Ship]      │
│                                                              │
│  Event-Driven Data Sync:                                    │
│  ┌────────┐   UserCreated   ┌────────┐                     │
│  │  User  │─────Event──────▶│ Order  │                     │
│  │Service │                 │Service │                     │
│  └────────┘                 └────────┘                     │
│                             (caches user info locally)     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Challenges

```
┌─────────────────────────────────────────────────────────────┐
│               Microservices Challenges                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Distributed System Complexity                          │
│     • Network failures                                      │
│     • Latency                                               │
│     • Partial failures                                      │
│     Solution: Circuit breakers, retries, timeouts          │
│                                                              │
│  2. Data Consistency                                        │
│     • No distributed transactions                          │
│     • Eventual consistency                                  │
│     Solution: Saga pattern, event sourcing                 │
│                                                              │
│  3. Service Discovery                                       │
│     • How do services find each other?                     │
│     Solution: Service registry (Consul, Eureka, K8s DNS)   │
│                                                              │
│  4. Observability                                           │
│     • Debugging across services                            │
│     • Finding bottlenecks                                  │
│     Solution: Distributed tracing, centralized logging     │
│                                                              │
│  5. Testing                                                 │
│     • Integration testing complexity                       │
│     • Contract testing                                      │
│     Solution: Consumer-driven contracts, service mocks     │
│                                                              │
│  6. Operational Overhead                                    │
│     • Many services to deploy/monitor                      │
│     • Configuration management                             │
│     Solution: Container orchestration (K8s), CI/CD         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### When NOT to Use Microservices

```
┌─────────────────────────────────────────────────────────────┐
│              When NOT to Use Microservices                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✗ Small team (< 10 developers)                            │
│  ✗ Simple domain                                           │
│  ✗ New product (domain not well understood)                │
│  ✗ Strong consistency requirements                         │
│  ✗ No DevOps/platform team                                 │
│                                                              │
│  "If you can't build a well-structured monolith,           │
│   what makes you think microservices are the answer?"      │
│                                           - Simon Brown     │
│                                                              │
│  Start with:                                                │
│  Modular Monolith → Extract services when needed           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What are microservices? How do they differ from monoliths?
2. What are the benefits and drawbacks of microservices?

### Intermediate
3. How do you decide service boundaries?
4. How do microservices communicate?

### Advanced
5. How would you handle transactions across microservices?
6. Design the migration from a monolith to microservices.

---

[Back to Design Patterns](../README.md) | [Next: Event-Driven Architecture →](../02-event-driven/README.md)
