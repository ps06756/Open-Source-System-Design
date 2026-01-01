# Microservices vs Monolith

Choosing between microservices and monolithic architecture is one of the most important architectural decisions. This chapter explores when to use each.

## Table of Contents
- [Monolithic Architecture](#monolithic-architecture)
- [Microservices Architecture](#microservices-architecture)
- [Comparison](#comparison)
- [Migration Strategies](#migration-strategies)
- [Key Takeaways](#key-takeaways)

## Monolithic Architecture

All components run as a single application in a single process.

```
┌────────────────────────────────────────────────────────┐
│                     Monolith                           │
│  ┌─────────────────────────────────────────────────┐  │
│  │                                                 │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │  │
│  │  │  User   │  │  Order  │  │ Product │        │  │
│  │  │ Module  │  │ Module  │  │ Module  │        │  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘        │  │
│  │       │            │            │              │  │
│  │       └────────────┼────────────┘              │  │
│  │                    │                           │  │
│  │             ┌──────┴──────┐                    │  │
│  │             │  Database   │                    │  │
│  │             └─────────────┘                    │  │
│  │                                                 │  │
│  └─────────────────────────────────────────────────┘  │
│                                                        │
│                   Single Deployment                    │
└────────────────────────────────────────────────────────┘
```

### Advantages

| Advantage | Description |
|-----------|-------------|
| Simplicity | One codebase, easy to understand |
| Easy development | No distributed systems complexity |
| Easy testing | Single process, straightforward tests |
| Easy deployment | One artifact to deploy |
| Low latency | In-process function calls |
| Strong consistency | Single database, ACID transactions |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Scaling | Must scale entire app, not just busy parts |
| Deployment risk | Change in one part affects everything |
| Technology lock-in | Stuck with one tech stack |
| Team scaling | Large teams step on each other |
| Long build times | Build and test entire application |

### When to Use Monolith

```
✓ Small team (< 10 developers)
✓ New project/startup (MVP)
✓ Well-defined, stable domain
✓ Simple scaling requirements
✓ Strong consistency requirements
```

## Microservices Architecture

Application as a collection of small, independent services.

```
┌────────────────────────────────────────────────────────┐
│                   Microservices                        │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ User Service │  │Order Service │  │Product Svc   │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │
│  │ │   API    │ │  │ │   API    │ │  │ │   API    │ │ │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │
│  │ │    DB    │ │  │ │    DB    │ │  │ │    DB    │ │ │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                        │
│         Each service independently deployable          │
└────────────────────────────────────────────────────────┘
```

### Advantages

| Advantage | Description |
|-----------|-------------|
| Independent scaling | Scale busy services only |
| Independent deployment | Deploy without affecting others |
| Technology diversity | Best tool for each job |
| Team autonomy | Teams own their services |
| Fault isolation | Failure in one doesn't crash all |
| Faster development | Smaller codebases |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| Distributed complexity | Network failures, latency |
| Data consistency | No ACID across services |
| Operational overhead | Many services to manage |
| Testing complexity | Integration tests are harder |
| Debugging difficulty | Traces span services |
| Initial overhead | More setup required |

### When to Use Microservices

```
✓ Large team (multiple teams)
✓ Complex, evolving domain
✓ Different scaling requirements per component
✓ Need for technology diversity
✓ Can invest in infrastructure
```

## Comparison

### Architecture Comparison

```
┌─────────────────────────────────────────────────────────┐
│                    Comparison                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Aspect           │ Monolith      │ Microservices      │
│  ─────────────────┼───────────────┼────────────────────│
│  Codebase         │ Single        │ Multiple           │
│  Deployment       │ All or nothing│ Independent        │
│  Scaling          │ Whole app     │ Per service        │
│  Data             │ Shared DB     │ DB per service     │
│  Communication    │ Function call │ Network (HTTP/gRPC)│
│  Consistency      │ ACID          │ Eventual           │
│  Team structure   │ One team      │ Multiple teams     │
│  Complexity       │ In code       │ In infrastructure  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Communication Patterns

```
Monolith:
┌──────────────────────────────────────┐
│  UserModule.getUser(id)              │
│  OrderModule.createOrder(user, items)│
│  PaymentModule.charge(order)         │
│                                      │
│  All in-process, nanoseconds        │
└──────────────────────────────────────┘

Microservices:
┌──────────────────────────────────────┐
│  GET http://user-service/users/123  │
│  POST http://order-service/orders    │
│  POST http://payment-service/charge  │
│                                      │
│  Network calls, milliseconds        │
└──────────────────────────────────────┘
```

### Data Management

```
Monolith (shared database):
┌─────────────────────────────────────────┐
│  BEGIN TRANSACTION                      │
│    UPDATE users SET ...                 │
│    INSERT INTO orders ...               │
│    UPDATE inventory ...                 │
│  COMMIT                                 │
│                                         │
│  ACID guaranteed                       │
└─────────────────────────────────────────┘

Microservices (database per service):
┌─────────────────────────────────────────┐
│  User Service: UPDATE users ...        │
│  Order Service: INSERT orders ...       │
│  Inventory Service: UPDATE inventory ...│
│                                         │
│  Saga pattern for consistency          │
│  Eventually consistent                  │
└─────────────────────────────────────────┘
```

## Migration Strategies

### Strangler Fig Pattern

Gradually replace parts of the monolith.

```
Phase 1: Extract one service
┌──────────────────┐     ┌──────────────┐
│    Monolith      │     │  New Service │
│  ┌────────────┐  │     │              │
│  │  Extracted │──┼────▶│  (Users)     │
│  │    ─────   │  │     │              │
│  └────────────┘  │     └──────────────┘
└──────────────────┘

Phase 2: Route traffic to new service
┌──────────────────┐     ┌──────────────┐
│    Monolith      │     │  User Service│
│                  │     │              │
│  (Users removed) │     │   Active     │
│                  │     │              │
└──────────────────┘     └──────────────┘

Repeat until monolith is gone
```

### Anti-Corruption Layer

Isolate new services from legacy code.

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌─────────────┐     ┌───────────────────┐            │
│  │   Legacy    │     │  Anti-Corruption  │            │
│  │  Monolith   │◀───▶│      Layer        │◀────────▶  │
│  │             │     │                   │     New    │
│  └─────────────┘     │  - Translates     │   Services │
│                      │  - Adapts         │            │
│                      │  - Isolates       │            │
│                      └───────────────────┘            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Branch by Abstraction

Create abstraction, implement new version behind it.

```
1. Create abstraction
   interface PaymentProcessor { ... }

2. Implement with legacy
   class LegacyPaymentProcessor implements PaymentProcessor

3. Implement new service
   class NewPaymentService implements PaymentProcessor

4. Switch implementations
   @Bean PaymentProcessor payment() {
     return featureFlag.enabled("new_payment")
       ? new NewPaymentService()
       : new LegacyPaymentProcessor();
   }
```

## Service Design Principles

### Single Responsibility

```
Bad: OrderService handles orders, payments, inventory, notifications

Good:
- OrderService: Order lifecycle
- PaymentService: Payment processing
- InventoryService: Stock management
- NotificationService: Sending emails/SMS
```

### Domain-Driven Design

```
┌────────────────────────────────────────────────────────┐
│             Bounded Contexts                           │
│                                                        │
│  ┌─────────────────┐   ┌─────────────────┐            │
│  │   Order Context │   │ Shipping Context│            │
│  │                 │   │                 │            │
│  │  Order          │   │  Shipment       │            │
│  │  OrderLine      │   │  Package        │            │
│  │  Customer (ref) │   │  Address        │            │
│  └─────────────────┘   └─────────────────┘            │
│                                                        │
│  Same entity (Customer) may have different            │
│  representations in different contexts                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### API Gateway

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Clients ──▶ API Gateway ──┬──▶ User Service          │
│                            ├──▶ Order Service         │
│                            ├──▶ Product Service       │
│                            └──▶ Payment Service       │
│                                                        │
│  Gateway handles:                                      │
│  - Routing                                            │
│  - Authentication                                      │
│  - Rate limiting                                      │
│  - Request aggregation                                │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Start with monolith** unless you have proven need for microservices
2. **Microservices = distributed complexity** - don't underestimate
3. **Team structure matters** - Conway's Law applies
4. **Data ownership** is key - each service owns its data
5. **Strangler Fig** for gradual migration
6. **Domain-Driven Design** helps find service boundaries

## Practice Questions

1. You're building a startup MVP. Which architecture would you choose?
2. Your 50-person team struggles with a monolith. How would you migrate?
3. How do you handle transactions across microservices?
4. What are the infrastructure requirements for microservices?

## Further Reading

- [Microservices Patterns](https://microservices.io/patterns/)
- [Building Microservices](https://samnewman.io/books/building_microservices_2nd_edition/)
- [Monolith First](https://martinfowler.com/bliki/MonolithFirst.html)

---

Next: [Event-Driven Architecture](../02-event-driven/README.md)
