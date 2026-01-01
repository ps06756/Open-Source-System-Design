# Interview Template

Use this as a checklist during your system design interview.

## Phase 1: Requirements (5-7 min)

### Functional Requirements
```
□ What are the core features?
□ Who are the users?
□ What can users do?
□ Any different user types?

Write down 3-5 key features:
1. _______________________
2. _______________________
3. _______________________
4. _______________________
5. _______________________
```

### Non-Functional Requirements
```
□ Scale: How many users? DAU?
□ Latency: What's acceptable?
□ Availability: What uptime needed?
□ Consistency: Strong or eventual?
□ Data retention: How long?

Key numbers:
- Users: _______
- Requests/sec: _______
- Latency target: _______
- Availability: _______
```

### Out of Scope
```
□ What are we NOT building?

Out of scope:
1. _______________________
2. _______________________
```

## Phase 2: Estimation (3-5 min)

### Traffic
```
DAU: _______
Actions/user/day: _______

Writes/day: DAU × writes = _______
Writes/sec: writes/day ÷ 86,400 = _______

Reads/day: DAU × reads = _______
Reads/sec: reads/day ÷ 86,400 = _______

Read:Write ratio: _______
```

### Storage
```
Data per item: _______
Items/day: _______
Retention: _______

Storage = items × size × days = _______

With replication (3x): _______
```

### Bandwidth
```
Ingress: writes/sec × size = _______
Egress: reads/sec × size = _______
```

## Phase 3: High-Level Design (10-15 min)

### Draw Architecture
```
┌────────────────────────────────────────────────────┐
│                                                    │
│  [Client]                                         │
│      │                                            │
│      ▼                                            │
│  [Load Balancer]                                  │
│      │                                            │
│      ▼                                            │
│  [API Servers] ──────────────────────────────────│
│      │                    │                       │
│      ▼                    ▼                       │
│  [Cache]              [Database]                  │
│                           │                       │
│                           ▼                       │
│                       [Storage]                   │
│                                                    │
└────────────────────────────────────────────────────┘

Notes:
_____________________________________________
_____________________________________________
```

### Define APIs
```
□ Core endpoints

API 1: _____ /_____
  Request: _____
  Response: _____

API 2: _____ /_____
  Request: _____
  Response: _____

API 3: _____ /_____
  Request: _____
  Response: _____
```

### Database Schema
```
□ Main entities and relationships

Table 1: _______
  - id (PK)
  - _______
  - _______
  - _______

Table 2: _______
  - id (PK)
  - _______ (FK)
  - _______
  - _______

Indexes:
  - _______
  - _______
```

## Phase 4: Deep Dive (15-20 min)

### Component 1: _______
```
Problem: _______________________

Options considered:
1. _______ (pros: _____, cons: _____)
2. _______ (pros: _____, cons: _____)

Chosen approach: _______

How it works:
1. _______
2. _______
3. _______

Edge cases:
- _______
- _______
```

### Component 2: _______
```
Problem: _______________________

Options considered:
1. _______ (pros: _____, cons: _____)
2. _______ (pros: _____, cons: _____)

Chosen approach: _______

How it works:
1. _______
2. _______
3. _______

Edge cases:
- _______
- _______
```

### Scaling Considerations
```
□ How to scale when 10x users?

Database: _______
Cache: _______
Application: _______
```

### Failure Handling
```
□ What if X fails?

Database fails: _______
Cache fails: _______
Service fails: _______
```

## Phase 5: Trade-offs (5-10 min)

### Key Trade-offs
```
Trade-off 1:
  Chose: _______
  Over: _______
  Because: _______

Trade-off 2:
  Chose: _______
  Over: _______
  Because: _______

Trade-off 3:
  Chose: _______
  Over: _______
  Because: _______
```

### What Would Change at 10x Scale?
```
1. _______________________
2. _______________________
3. _______________________
```

## Checklist Before Ending

```
□ Did I clarify requirements?
□ Did I estimate scale?
□ Did I draw a clear diagram?
□ Did I define APIs?
□ Did I discuss database schema?
□ Did I consider caching?
□ Did I address scaling?
□ Did I discuss failure handling?
□ Did I explain trade-offs?
□ Did I leave time for questions?
```

## Quick Reference

### Components to Include
```
□ Load Balancer
□ API Gateway
□ Application Servers
□ Database (+ replicas)
□ Cache (Redis/Memcached)
□ CDN (if applicable)
□ Message Queue (if applicable)
□ Blob Storage (if applicable)
□ Search (if applicable)
```

### Database Decision
```
□ SQL: Transactions, complex queries, relationships
□ NoSQL: Scale, flexible schema, simple access
□ Cache: Hot data, session, real-time
```

### Scaling Strategies
```
□ Horizontal: More servers
□ Vertical: Bigger servers
□ Caching: Reduce DB load
□ Sharding: Split database
□ Replication: Read replicas
□ CDN: Static content
□ Async: Message queues
```

---

[Back to Resources](./README.md)
