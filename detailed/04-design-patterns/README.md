# Module 4: Design Patterns

Architectural patterns that solve common distributed systems problems at scale.

## Chapters

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Microservices](./01-microservices/README.md) | Service decomposition, boundaries, communication |
| 2 | [Event-Driven Architecture](./02-event-driven/README.md) | Events, choreography vs orchestration |
| 3 | [CQRS & Event Sourcing](./03-cqrs-event-sourcing/README.md) | Separating reads/writes, event logs |
| 4 | [Saga Pattern](./04-saga-pattern/README.md) | Distributed transactions, compensation |
| 5 | [Circuit Breaker](./05-circuit-breaker/README.md) | Fault tolerance, graceful degradation |
| 6 | [Sidecar & Service Mesh](./06-sidecar-service-mesh/README.md) | Cross-cutting concerns, Istio/Envoy |

## Pattern Selection Guide

```
┌─────────────────────────────────────────────────────────────┐
│               When to Use Each Pattern                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Microservices                                               │
│  └─ Large teams, independent deployments, scaling needs    │
│                                                              │
│  Event-Driven                                                │
│  └─ Loose coupling, real-time processing, async workflows  │
│                                                              │
│  CQRS                                                        │
│  └─ Different read/write patterns, complex queries         │
│                                                              │
│  Event Sourcing                                              │
│  └─ Audit trails, temporal queries, debugging              │
│                                                              │
│  Saga                                                        │
│  └─ Distributed transactions across services               │
│                                                              │
│  Circuit Breaker                                             │
│  └─ External dependencies, fault tolerance                 │
│                                                              │
│  Service Mesh                                                │
│  └─ Complex networking, observability, security            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

[← Previous Module: Building Blocks](../03-building-blocks/README.md) | [Next Module: System Designs →](../05-system-designs/README.md)
