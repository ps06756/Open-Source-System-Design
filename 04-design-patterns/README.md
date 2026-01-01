# Module 4: Design Patterns

Architectural patterns are reusable solutions to common problems in distributed systems. Understanding when and how to apply these patterns is crucial for system design.

## Topics

1. [Microservices vs Monolith](01-microservices-monolith/README.md) - When to use each
2. [Event-Driven Architecture](02-event-driven/README.md) - Async, event sourcing
3. [CQRS](03-cqrs/README.md) - Separating reads and writes
4. [Saga Pattern](04-saga/README.md) - Distributed transactions
5. [Circuit Breaker](05-circuit-breaker/README.md) - Fault tolerance
6. [Sidecar & Ambassador](06-sidecar-ambassador/README.md) - Service mesh patterns

## Why Patterns Matter

Patterns provide:
- **Proven solutions** to recurring problems
- **Common vocabulary** for discussing architectures
- **Trade-off awareness** for better decisions
- **Interview readiness** - frequently asked about

## Pattern Selection Guide

| Problem | Pattern |
|---------|---------|
| Growing monolith | Microservices |
| Decoupling services | Event-Driven |
| Read/write scale mismatch | CQRS |
| Distributed transactions | Saga |
| Cascading failures | Circuit Breaker |
| Cross-cutting concerns | Sidecar |

## Prerequisites

Complete [Module 3: Building Blocks](../03-building-blocks/README.md) first.

## Next Steps

After this module, proceed to [Module 5: System Designs](../05-system-designs/README.md).
