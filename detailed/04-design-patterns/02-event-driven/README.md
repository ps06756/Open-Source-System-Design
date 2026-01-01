# Event-Driven Architecture

Event-Driven Architecture (EDA) is a design pattern where the flow of the program is determined by events—significant changes in state that the system cares about.

## Table of Contents
1. [What is Event-Driven Architecture?](#what-is-event-driven-architecture)
2. [Event Types](#event-types)
3. [Choreography vs Orchestration](#choreography-vs-orchestration)
4. [Event Design](#event-design)
5. [Benefits and Challenges](#benefits-and-challenges)
6. [Interview Questions](#interview-questions)

---

## What is Event-Driven Architecture?

```
┌─────────────────────────────────────────────────────────────┐
│              Event-Driven Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traditional Request-Response:                              │
│  ┌────────┐  request   ┌────────┐  request   ┌────────┐    │
│  │ Order  │───────────▶│Inventory│──────────▶│Shipping│    │
│  │Service │◀───────────│Service │◀──────────│Service │    │
│  └────────┘  response  └────────┘  response  └────────┘    │
│  Tight coupling: Order knows about Inventory and Shipping  │
│                                                              │
│  Event-Driven:                                               │
│  ┌────────┐  OrderPlaced  ┌─────────────────────────────┐  │
│  │ Order  │──────────────▶│       Event Bus/Broker      │  │
│  │Service │               │                             │  │
│  └────────┘               └─────────────────────────────┘  │
│                                  │           │              │
│                                  ▼           ▼              │
│                           ┌────────┐   ┌────────┐          │
│                           │Inventory│  │Shipping│          │
│                           │Service │   │Service │          │
│                           └────────┘   └────────┘          │
│  Loose coupling: Order just publishes, doesn't know who    │
│  consumes the event                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    Core Concepts                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Event: A significant change in state                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  {                                                      │ │
│  │    "eventType": "OrderPlaced",                         │ │
│  │    "eventId": "uuid-123",                              │ │
│  │    "timestamp": "2024-01-15T10:30:00Z",               │ │
│  │    "data": {                                           │ │
│  │      "orderId": "order-456",                          │ │
│  │      "customerId": "cust-789",                        │ │
│  │      "items": [...],                                  │ │
│  │      "total": 99.99                                   │ │
│  │    }                                                    │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Producer: Service that publishes events                   │
│  Consumer: Service that subscribes to and handles events   │
│  Event Broker: Infrastructure that routes events           │
│    (Kafka, RabbitMQ, AWS EventBridge, etc.)               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Event Types

```
┌─────────────────────────────────────────────────────────────┐
│                      Event Types                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Domain Events (Business Events)                        │
│     What happened in the business domain                   │
│     ┌────────────────────────────────────────────────────┐ │
│     │ • OrderPlaced                                       │ │
│     │ • PaymentReceived                                   │ │
│     │ • ItemShipped                                       │ │
│     │ • UserRegistered                                    │ │
│     └────────────────────────────────────────────────────┘ │
│     Past tense! Immutable facts that happened.            │
│                                                              │
│  2. Integration Events                                      │
│     For communication between bounded contexts             │
│     May be transformed/filtered from domain events         │
│                                                              │
│  3. Notification Events (Thin Events)                      │
│     ┌────────────────────────────────────────────────────┐ │
│     │ { "type": "OrderPlaced", "orderId": "123" }        │ │
│     └────────────────────────────────────────────────────┘ │
│     Just notify something happened; consumer fetches data │
│     ✓ Smaller payloads                                    │
│     ✗ Requires callback to producer                      │
│                                                              │
│  4. Event-Carried State Transfer (Fat Events)              │
│     ┌────────────────────────────────────────────────────┐ │
│     │ { "type": "OrderPlaced", "order": { ...full... } } │ │
│     └────────────────────────────────────────────────────┘ │
│     Contains all data consumer needs                       │
│     ✓ Consumer is self-sufficient                         │
│     ✗ Larger payloads, potential staleness               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Choreography vs Orchestration

```
┌─────────────────────────────────────────────────────────────┐
│              Choreography vs Orchestration                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Choreography (Decentralized):                             │
│  Each service knows when to act based on events            │
│                                                              │
│  ┌────────┐  OrderPlaced  ┌────────┐  InventoryReserved   │
│  │ Order  │──────────────▶│Inventory│─────────────────┐   │
│  │Service │               │Service │                   │   │
│  └────────┘               └────────┘                   │   │
│                                                         ▼   │
│                           ┌────────┐  PaymentCharged  ┌──┴─┐│
│                           │Shipping│◀─────────────────│Pay ││
│                           │Service │                  │ment││
│                           └────────┘                  └────┘│
│                                                              │
│  ✓ Loose coupling                                          │
│  ✓ Services are autonomous                                 │
│  ✗ Hard to see full workflow                              │
│  ✗ Can become "event spaghetti"                           │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  Orchestration (Centralized):                              │
│  A central coordinator controls the workflow               │
│                                                              │
│       ┌──────────────────────────────────────────┐         │
│       │            Order Orchestrator            │         │
│       │  1. Reserve Inventory                    │         │
│       │  2. Charge Payment                       │         │
│       │  3. Initiate Shipping                    │         │
│       └────────┬─────────┬──────────┬────────────┘         │
│                │         │          │                       │
│                ▼         ▼          ▼                       │
│           ┌────────┐ ┌────────┐ ┌────────┐                 │
│           │Inventory│ │Payment │ │Shipping│                 │
│           │Service │ │Service │ │Service │                 │
│           └────────┘ └────────┘ └────────┘                 │
│                                                              │
│  ✓ Clear workflow visibility                               │
│  ✓ Easier error handling                                   │
│  ✗ Orchestrator can become bottleneck                     │
│  ✗ More coupling through orchestrator                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Which?

```
┌─────────────────────────────────────────────────────────────┐
│                  When to Use Which?                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Use Choreography When:                                     │
│  • Simple workflows (2-3 steps)                            │
│  • Services truly independent                              │
│  • Loose coupling is priority                              │
│  • Multiple consumers per event                            │
│  Example: User signup → send email, create profile         │
│                                                              │
│  Use Orchestration When:                                    │
│  • Complex workflows with many steps                       │
│  • Workflow visibility is important                        │
│  • Error handling/compensation is complex                  │
│  • Business process needs to be explicit                   │
│  Example: Order fulfillment, loan approval                 │
│                                                              │
│  Hybrid Approach:                                           │
│  • Orchestration within bounded context                    │
│  • Choreography between bounded contexts                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Event Design

```
┌─────────────────────────────────────────────────────────────┐
│                    Event Design                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Event Schema:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  {                                                      │ │
│  │    // Metadata                                          │ │
│  │    "eventId": "uuid",           // Unique ID           │ │
│  │    "eventType": "OrderPlaced",  // What happened       │ │
│  │    "timestamp": "ISO8601",      // When                │ │
│  │    "version": "1.0",            // Schema version      │ │
│  │    "source": "order-service",   // Who published       │ │
│  │    "correlationId": "uuid",     // For tracing         │ │
│  │                                                         │ │
│  │    // Payload                                           │ │
│  │    "data": {                                           │ │
│  │      "orderId": "...",                                 │ │
│  │      "customerId": "...",                              │ │
│  │      ...                                               │ │
│  │    }                                                    │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Best Practices:                                            │
│  • Events are immutable (never update, only append)        │
│  • Use past tense (OrderPlaced, not PlaceOrder)           │
│  • Include enough context for consumers                    │
│  • Version your schema                                     │
│  • Use schema registry (Avro, Protobuf, JSON Schema)      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Ordering and Idempotency

```
┌─────────────────────────────────────────────────────────────┐
│            Event Ordering and Idempotency                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Ordering Challenges:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Events published:     Events received:                │ │
│  │  1. OrderPlaced        1. OrderPlaced  ✓              │ │
│  │  2. OrderUpdated       3. OrderCancelled (!)          │ │
│  │  3. OrderCancelled     2. OrderUpdated  (!)           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Solutions:                                                  │
│  • Partition by entity ID (all order events to same queue)│
│  • Include sequence numbers                                │
│  • Event timestamp + conflict resolution                   │
│                                                              │
│  Idempotency (handling duplicates):                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Problem: Same event delivered twice                   │ │
│  │  OrderPlaced → charge $100                             │ │
│  │  OrderPlaced → charge $100 (duplicate!)               │ │
│  │                                                         │ │
│  │  Solution: Track processed event IDs                   │ │
│  │  if eventId not in processed_events:                  │ │
│  │      process(event)                                    │ │
│  │      processed_events.add(eventId)                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Benefits and Challenges

```
┌─────────────────────────────────────────────────────────────┐
│                Benefits and Challenges                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Benefits:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✓ Loose Coupling                                       │ │
│  │   Producer doesn't know/care about consumers           │ │
│  │                                                         │ │
│  │ ✓ Scalability                                          │ │
│  │   Add consumers independently                          │ │
│  │                                                         │ │
│  │ ✓ Resilience                                           │ │
│  │   Events persisted; consumers can catch up             │ │
│  │                                                         │ │
│  │ ✓ Extensibility                                        │ │
│  │   Add new consumers without changing producers         │ │
│  │                                                         │ │
│  │ ✓ Audit Trail                                          │ │
│  │   Events provide natural history                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Challenges:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✗ Eventual Consistency                                 │ │
│  │   Data won't be immediately consistent                 │ │
│  │                                                         │ │
│  │ ✗ Debugging Complexity                                 │ │
│  │   Hard to trace flow across services                   │ │
│  │                                                         │ │
│  │ ✗ Event Ordering                                       │ │
│  │   May receive events out of order                      │ │
│  │                                                         │ │
│  │ ✗ Error Handling                                       │ │
│  │   Failed events need retry/dead letter queues          │ │
│  │                                                         │ │
│  │ ✗ Testing                                              │ │
│  │   Integration tests are complex                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is event-driven architecture?
2. What is the difference between a command and an event?

### Intermediate
3. Compare choreography vs orchestration.
4. How do you handle event ordering?

### Advanced
5. How would you migrate from a request-response architecture to event-driven?
6. Design an event-driven order processing system.

---

[← Previous: Microservices](../01-microservices/README.md) | [Next: CQRS & Event Sourcing →](../03-cqrs-event-sourcing/README.md)
