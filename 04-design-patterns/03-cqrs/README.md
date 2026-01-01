# CQRS (Command Query Responsibility Segregation)

CQRS separates read and write operations into different models, optimizing each independently.

## Table of Contents
- [What is CQRS?](#what-is-cqrs)
- [When to Use CQRS](#when-to-use-cqrs)
- [Implementation](#implementation)
- [CQRS with Event Sourcing](#cqrs-with-event-sourcing)
- [Key Takeaways](#key-takeaways)

## What is CQRS?

Traditional architecture uses the same model for reads and writes. CQRS separates them.

```
Traditional (Single Model):
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌─────────┐                                          │
│  │  Client │                                          │
│  └────┬────┘                                          │
│       │                                                │
│       ▼                                                │
│  ┌─────────────────────────────────┐                  │
│  │         Application              │                  │
│  │  ┌───────────────────────────┐  │                  │
│  │  │     Single Model          │  │                  │
│  │  │   (reads and writes)      │  │                  │
│  │  └───────────────────────────┘  │                  │
│  └─────────────────────────────────┘                  │
│                    │                                   │
│                    ▼                                   │
│            ┌──────────────┐                           │
│            │   Database   │                           │
│            └──────────────┘                           │
│                                                        │
└────────────────────────────────────────────────────────┘

CQRS (Separate Models):
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌─────────┐                    ┌─────────┐           │
│  │ Client  │                    │ Client  │           │
│  │ (write) │                    │ (read)  │           │
│  └────┬────┘                    └────┬────┘           │
│       │                              │                 │
│       ▼                              ▼                 │
│  ┌──────────────┐           ┌──────────────┐          │
│  │   Command    │           │    Query     │          │
│  │    Model     │           │    Model     │          │
│  │  (writes)    │           │   (reads)    │          │
│  └──────┬───────┘           └──────┬───────┘          │
│         │                          │                   │
│         ▼                          ▼                   │
│  ┌──────────────┐           ┌──────────────┐          │
│  │ Write Store  │──────────▶│ Read Store   │          │
│  └──────────────┘   sync    └──────────────┘          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| Command | Request to change state (CreateOrder) |
| Query | Request to read state (GetOrderById) |
| Command Model | Optimized for writes, enforces business rules |
| Query Model | Optimized for reads, denormalized for fast queries |

## When to Use CQRS

### Good Fit

```
✓ Read/write ratio is heavily skewed (1000:1 reads)
✓ Complex domain with rich business logic
✓ Different scaling requirements for reads vs writes
✓ Complex queries that don't match write model
✓ Team separation (read team vs write team)
✓ Used with Event Sourcing
```

### Not a Good Fit

```
✗ Simple CRUD applications
✗ Small teams / simple domains
✗ Strong consistency requirements
✗ Low traffic where optimization isn't needed
```

### Benefits and Trade-offs

| Benefits | Trade-offs |
|----------|------------|
| Independent scaling | Increased complexity |
| Optimized models | Eventual consistency |
| Simpler queries | More code to maintain |
| Performance | Sync between models |

## Implementation

### Commands and Queries

```python
# Commands (change state)
@dataclass
class CreateOrderCommand:
    customer_id: str
    items: List[OrderItem]

@dataclass
class CancelOrderCommand:
    order_id: str
    reason: str

# Queries (read state)
@dataclass
class GetOrderByIdQuery:
    order_id: str

@dataclass
class GetOrdersByCustomerQuery:
    customer_id: str
    status: Optional[str]
```

### Command Handler

```python
class CreateOrderHandler:
    def __init__(self, order_repository, event_publisher):
        self.order_repository = order_repository
        self.event_publisher = event_publisher

    def handle(self, command: CreateOrderCommand):
        # Business logic and validation
        order = Order.create(
            customer_id=command.customer_id,
            items=command.items
        )

        # Persist to write store
        self.order_repository.save(order)

        # Publish event for read model sync
        self.event_publisher.publish(
            OrderCreatedEvent(
                order_id=order.id,
                customer_id=order.customer_id,
                items=order.items,
                total=order.total
            )
        )

        return order.id
```

### Query Handler

```python
class GetOrdersByCustomerHandler:
    def __init__(self, read_repository):
        self.read_repository = read_repository

    def handle(self, query: GetOrdersByCustomerQuery):
        # Simple read from optimized read store
        return self.read_repository.find_by_customer(
            customer_id=query.customer_id,
            status=query.status
        )
```

### Read Model Synchronization

```
┌────────────────────────────────────────────────────────┐
│               Read Model Sync                          │
│                                                        │
│  Write Store                     Read Store            │
│  ┌──────────────┐               ┌──────────────┐      │
│  │  Normalized  │               │ Denormalized │      │
│  │              │               │              │      │
│  │ - Orders     │    Events     │ - OrderView  │      │
│  │ - OrderItems │──────────────▶│   (flat)     │      │
│  │ - Customers  │               │              │      │
│  └──────────────┘               └──────────────┘      │
│                                                        │
│  Projection rebuilds read model from events           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

```python
class OrderProjection:
    """Builds read model from events"""

    def __init__(self, read_store):
        self.read_store = read_store

    def handle_order_created(self, event: OrderCreatedEvent):
        view = OrderView(
            order_id=event.order_id,
            customer_id=event.customer_id,
            customer_name=self.lookup_customer_name(event.customer_id),
            items=event.items,
            total=event.total,
            status="created",
            created_at=event.timestamp
        )
        self.read_store.insert(view)

    def handle_order_shipped(self, event: OrderShippedEvent):
        self.read_store.update(
            order_id=event.order_id,
            status="shipped",
            shipped_at=event.timestamp,
            tracking_number=event.tracking_number
        )
```

### Database Separation

```
┌────────────────────────────────────────────────────────┐
│            Database Separation Options                 │
│                                                        │
│  Option 1: Same database, different tables            │
│  ┌─────────────────────────────────────────┐          │
│  │  PostgreSQL                              │          │
│  │  ├── orders (write)                     │          │
│  │  ├── order_items (write)                │          │
│  │  └── order_views (read)                 │          │
│  └─────────────────────────────────────────┘          │
│                                                        │
│  Option 2: Different databases                        │
│  ┌──────────────┐      ┌──────────────┐               │
│  │  PostgreSQL  │      │ Elasticsearch│               │
│  │   (writes)   │ ───▶ │   (reads)    │               │
│  └──────────────┘      └──────────────┘               │
│                                                        │
│  Option 3: Different technologies                     │
│  ┌──────────────┐      ┌──────────────┐               │
│  │  PostgreSQL  │ ───▶ │    Redis     │               │
│  │   (writes)   │      │  (hot data)  │               │
│  └──────────────┘      └──────────────┘               │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## CQRS with Event Sourcing

CQRS and Event Sourcing are often used together.

```
┌────────────────────────────────────────────────────────┐
│            CQRS + Event Sourcing                       │
│                                                        │
│  Commands ──▶ ┌──────────────┐                        │
│               │ Command Side │                        │
│               │              │                        │
│               │ Aggregate    │                        │
│               │    ↓         │                        │
│               │ Events       │                        │
│               └──────┬───────┘                        │
│                      │                                 │
│                      ▼                                 │
│               ┌──────────────┐                        │
│               │ Event Store  │                        │
│               │ (immutable)  │                        │
│               └──────┬───────┘                        │
│                      │                                 │
│                      ▼                                 │
│               ┌──────────────┐                        │
│               │ Projections  │                        │
│               │              │                        │
│               └──────┬───────┘                        │
│         ┌────────────┼────────────┐                   │
│         ▼            ▼            ▼                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │Read Model│ │Read Model│ │Read Model│◀── Queries  │
│  │    A     │ │    B     │ │    C     │              │
│  └──────────┘ └──────────┘ └──────────┘              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Multiple Read Models

```python
# Different read models for different use cases

# 1. Order list view (dashboard)
class OrderListView:
    order_id: str
    customer_name: str
    total: Decimal
    status: str
    created_at: datetime

# 2. Order detail view (full info)
class OrderDetailView:
    order_id: str
    customer: CustomerInfo
    items: List[OrderItemInfo]
    shipping: ShippingInfo
    payment: PaymentInfo
    history: List[StatusChange]

# 3. Analytics view (reporting)
class OrderAnalyticsView:
    date: date
    total_orders: int
    total_revenue: Decimal
    avg_order_value: Decimal
    by_category: Dict[str, int]
```

## Eventual Consistency

### Handling Consistency

```
┌────────────────────────────────────────────────────────┐
│           Eventual Consistency Strategies              │
│                                                        │
│  1. UI shows pending state                            │
│     "Your order is being processed..."               │
│                                                        │
│  2. Polling/WebSocket for updates                     │
│     Client polls until read model is updated          │
│                                                        │
│  3. Return write model data                           │
│     After command, return data from write side        │
│                                                        │
│  4. Synchronous read model update                     │
│     Update in same transaction (simple CQRS)         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Separate read and write models** for different optimizations
2. **Commands change state**, queries read state
3. **Read models are denormalized** for fast queries
4. **Accept eventual consistency** or use strategies to handle it
5. **Pairs well with Event Sourcing** for complex domains
6. **Adds complexity** - use only when benefits outweigh costs

## Practice Questions

1. Design a CQRS system for an e-commerce product catalog.
2. How do you handle the read model being out of sync?
3. When would you use separate databases for read/write?
4. Compare CQRS with traditional CRUD for a blog platform.

## Further Reading

- [CQRS by Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [CQRS Journey](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10))
- [Event Sourcing and CQRS](https://www.eventstore.com/cqrs-pattern)

---

Next: [Saga Pattern](../04-saga/README.md)
