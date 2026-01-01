# Event-Driven Architecture

Event-driven architecture (EDA) is a design pattern where services communicate by producing and consuming events.

## Table of Contents
- [What is Event-Driven Architecture?](#what-is-event-driven-architecture)
- [Event Types](#event-types)
- [Patterns](#patterns)
- [Event Sourcing](#event-sourcing)
- [Implementation](#implementation)
- [Key Takeaways](#key-takeaways)

## What is Event-Driven Architecture?

In EDA, services communicate through events - records of something that happened.

```
Traditional (Synchronous):
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Order   │────▶│ Inventory│────▶│ Shipping │
│ Service  │◀────│ Service  │◀────│ Service  │
└──────────┘     └──────────┘     └──────────┘
       Coupled, blocking calls

Event-Driven (Asynchronous):
┌──────────┐     ┌────────────────────────────────┐
│  Order   │────▶│          Event Bus             │
│ Service  │     │  ┌───────────────────────────┐ │
└──────────┘     │  │    OrderCreated Event     │ │
                 │  └───────────────────────────┘ │
                 └────────────────┬───────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │Inventory │ │ Shipping │ │  Email   │
              │ Service  │ │ Service  │ │ Service  │
              └──────────┘ └──────────┘ └──────────┘
       Decoupled, async processing
```

### Benefits

| Benefit | Description |
|---------|-------------|
| Loose coupling | Services don't know about each other |
| Scalability | Consumers can scale independently |
| Resilience | Producers continue if consumers fail |
| Flexibility | Add new consumers without changing producers |
| Audit trail | Events provide natural history |

### Challenges

| Challenge | Description |
|-----------|-------------|
| Complexity | Harder to trace and debug |
| Eventual consistency | No immediate confirmation |
| Event ordering | Order may not be guaranteed |
| Duplicate handling | Must design for idempotency |

## Event Types

### Domain Events

Represent something that happened in the business domain.

```json
{
  "eventType": "OrderPlaced",
  "eventId": "evt-123",
  "timestamp": "2024-01-15T10:30:00Z",
  "aggregateId": "order-456",
  "data": {
    "orderId": "order-456",
    "customerId": "cust-789",
    "items": [...],
    "total": 99.99
  }
}
```

### Integration Events

For communication between bounded contexts/services.

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  Order Context                Shipping Context         │
│  ┌─────────────────┐         ┌─────────────────┐      │
│  │ OrderPlaced     │         │ ShipmentCreated │      │
│  │ (domain event)  │         │ (domain event)  │      │
│  └────────┬────────┘         └─────────────────┘      │
│           │                          ▲                 │
│           ▼                          │                 │
│  ┌─────────────────┐                 │                 │
│  │ OrderSubmitted  │─────────────────┘                 │
│  │(integration evt)│                                   │
│  └─────────────────┘                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Patterns

### Event Notification

Simple notification that something happened. Consumer fetches details.

```
┌────────────────────────────────────────────────────────┐
│                 Event Notification                     │
│                                                        │
│  1. Producer publishes: { "orderId": "123" }          │
│  2. Consumer receives notification                     │
│  3. Consumer calls API: GET /orders/123               │
│  4. Consumer processes full order data                │
│                                                        │
│  Pros: Small events, current data                     │
│  Cons: Extra API call, producer must handle load      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Event-Carried State Transfer

Event contains all the data needed.

```
┌────────────────────────────────────────────────────────┐
│            Event-Carried State Transfer                │
│                                                        │
│  Event contains full data:                            │
│  {                                                     │
│    "eventType": "OrderPlaced",                        │
│    "order": {                                         │
│      "id": "123",                                     │
│      "customer": {...},                               │
│      "items": [...],                                  │
│      "total": 99.99                                   │
│    }                                                   │
│  }                                                     │
│                                                        │
│  Pros: No API calls, works offline                   │
│  Cons: Larger events, potentially stale data         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Choreography vs Orchestration

```
Choreography (decentralized):
┌────────┐ ──event──▶ ┌────────┐ ──event──▶ ┌────────┐
│ Order  │            │Inventory│            │Shipping│
│Service │ ◀──event── │Service │ ◀──event── │Service │
└────────┘            └────────┘            └────────┘
Each service knows what to do when it receives an event

Orchestration (centralized):
                ┌─────────────┐
                │ Orchestrator│
                │   (Saga)    │
                └──────┬──────┘
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌────────┐    ┌────────┐    ┌────────┐
   │ Order  │    │Inventory│   │Shipping│
   │Service │    │Service │    │Service │
   └────────┘    └────────┘    └────────┘
Orchestrator tells each service what to do
```

## Event Sourcing

Store all changes as a sequence of events, not just current state.

```
┌────────────────────────────────────────────────────────┐
│                   Event Sourcing                       │
│                                                        │
│  Traditional: Store current state                     │
│  ┌─────────────────────────────────────────┐          │
│  │ Account: balance = $150                  │          │
│  └─────────────────────────────────────────┘          │
│                                                        │
│  Event Sourced: Store all events                      │
│  ┌─────────────────────────────────────────┐          │
│  │ 1. AccountCreated(id=123)               │          │
│  │ 2. MoneyDeposited(amount=100)           │          │
│  │ 3. MoneyDeposited(amount=100)           │          │
│  │ 4. MoneyWithdrawn(amount=50)            │          │
│  └─────────────────────────────────────────┘          │
│                                                        │
│  Current state = replay(events)                       │
│  Balance = 0 + 100 + 100 - 50 = $150                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Event Store

```
┌────────────────────────────────────────────────────────┐
│                    Event Store                         │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Stream: account-123                             │   │
│  ├─────────┬──────────────────────┬───────────────┤   │
│  │ Version │ Event Type           │ Data          │   │
│  ├─────────┼──────────────────────┼───────────────┤   │
│  │ 1       │ AccountCreated       │ {owner: "Jo"} │   │
│  │ 2       │ MoneyDeposited       │ {amount: 100} │   │
│  │ 3       │ MoneyWithdrawn       │ {amount: 50}  │   │
│  └─────────┴──────────────────────┴───────────────┘   │
│                                                        │
│  Append-only, immutable                               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Snapshots

Optimization for aggregates with many events.

```
┌────────────────────────────────────────────────────────┐
│                     Snapshots                          │
│                                                        │
│  Without snapshots:                                    │
│  Load 10,000 events → replay all → current state     │
│                                                        │
│  With snapshots:                                       │
│  Load snapshot (event 9,900) + 100 events → state    │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Events: 1...9900 │ Snapshot │ Events: 9901-10000│  │
│  │   (skip these)   │  (load)  │    (replay)       │  │
│  └────────────────────────────────────────────────┘   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Benefits of Event Sourcing

| Benefit | Description |
|---------|-------------|
| Complete audit trail | Every change is recorded |
| Time travel | Rebuild state at any point in time |
| Debug production issues | Replay events to reproduce bugs |
| Event replay | Rebuild read models, fix bugs retroactively |
| Analytics | Mine event history for insights |

### Challenges

| Challenge | Description |
|-----------|-------------|
| Complexity | Different mental model |
| Event schema evolution | Changing event structure over time |
| Eventual consistency | Read models are behind |
| Storage growth | Events accumulate forever |

## Implementation

### Event Structure

```python
@dataclass
class Event:
    event_id: str
    event_type: str
    aggregate_id: str
    aggregate_type: str
    version: int
    timestamp: datetime
    data: dict
    metadata: dict

# Example
OrderPlaced(
    event_id="evt-uuid",
    event_type="OrderPlaced",
    aggregate_id="order-123",
    aggregate_type="Order",
    version=1,
    timestamp=datetime.now(),
    data={
        "customer_id": "cust-456",
        "items": [{"product_id": "prod-789", "quantity": 2}],
        "total": 99.99
    },
    metadata={
        "correlation_id": "req-abc",
        "user_id": "user-123"
    }
)
```

### Event Handler

```python
class InventoryEventHandler:
    def handle(self, event: Event):
        if event.event_type == "OrderPlaced":
            self.reserve_inventory(event.data)
        elif event.event_type == "OrderCancelled":
            self.release_inventory(event.data)

    def reserve_inventory(self, data):
        for item in data["items"]:
            inventory.reserve(
                product_id=item["product_id"],
                quantity=item["quantity"]
            )
```

### Idempotency

Handle duplicate events gracefully.

```python
class EventHandler:
    def handle(self, event: Event):
        # Check if already processed
        if self.event_store.is_processed(event.event_id):
            return  # Skip duplicate

        # Process event
        self.process(event)

        # Mark as processed
        self.event_store.mark_processed(event.event_id)
```

## Key Takeaways

1. **Events decouple** services effectively
2. **Choose pattern based on needs** - notification vs state transfer
3. **Event sourcing** provides complete history but adds complexity
4. **Design for idempotency** - duplicates will happen
5. **Choreography for simple flows**, orchestration for complex
6. **Start simple** - don't over-engineer

## Practice Questions

1. Design an event-driven order processing system.
2. When would you choose event sourcing over traditional storage?
3. How do you handle event schema changes?
4. Compare choreography vs orchestration for a payment flow.

## Further Reading

- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Domain-Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)

---

Next: [CQRS](../03-cqrs/README.md)
