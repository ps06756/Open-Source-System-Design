# CQRS & Event Sourcing

Two powerful patterns often used together: CQRS separates read and write models, while Event Sourcing stores state as a sequence of events.

## Table of Contents
1. [CQRS Pattern](#cqrs-pattern)
2. [Event Sourcing](#event-sourcing)
3. [CQRS + Event Sourcing](#cqrs--event-sourcing)
4. [Implementation Considerations](#implementation-considerations)
5. [Interview Questions](#interview-questions)

---

## CQRS Pattern

```
┌─────────────────────────────────────────────────────────────┐
│       CQRS - Command Query Responsibility Segregation       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traditional CRUD:                                           │
│  ┌────────┐    ┌─────────────┐    ┌──────────┐             │
│  │ Client │───▶│   Service   │───▶│ Database │             │
│  └────────┘    │ (Read+Write)│    │ (Single) │             │
│                └─────────────┘    └──────────┘             │
│  Same model for reads and writes                            │
│                                                              │
│  CQRS:                                                       │
│  ┌────────┐    Commands    ┌─────────────┐                 │
│  │        │───────────────▶│   Write     │──▶┌──────────┐  │
│  │        │                │   Model     │   │ Write DB │  │
│  │ Client │                └─────────────┘   └──────────┘  │
│  │        │                                       │         │
│  │        │                                  Sync │         │
│  │        │                                       ▼         │
│  │        │    Queries     ┌─────────────┐   ┌──────────┐  │
│  │        │◀───────────────│   Read      │◀──│ Read DB  │  │
│  └────────┘                │   Model     │   │ (Views)  │  │
│                            └─────────────┘   └──────────┘  │
│                                                              │
│  Commands: Change state (CreateOrder, UpdateUser)          │
│  Queries: Read state (GetOrder, ListUsers)                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why CQRS?

```
┌─────────────────────────────────────────────────────────────┐
│                       Why CQRS?                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Read and write patterns differ                   │
│                                                              │
│  Writes:                     Reads:                         │
│  ┌─────────────────────┐    ┌─────────────────────┐        │
│  │ • Validate business │    │ • Join multiple     │        │
│  │   rules             │    │   tables            │        │
│  │ • Enforce invariants│    │ • Aggregate data    │        │
│  │ • Single entity     │    │ • Different views   │        │
│  │   updates          │    │   for different     │        │
│  │ • Normalized data   │    │   clients          │        │
│  │ • Lower frequency   │    │ • High frequency    │        │
│  └─────────────────────┘    └─────────────────────┘        │
│                                                              │
│  Benefits of CQRS:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✓ Optimize read/write independently                   │ │
│  │ ✓ Scale reads and writes separately                   │ │
│  │ ✓ Simpler models (each does one thing well)          │ │
│  │ ✓ Different storage for each (SQL write, NoSQL read) │ │
│  │ ✓ Better for complex domains                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Trade-offs:                                                 │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✗ Increased complexity                                │ │
│  │ ✗ Eventual consistency between read/write             │ │
│  │ ✗ More infrastructure to maintain                     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### CQRS Example

```
┌─────────────────────────────────────────────────────────────┐
│                    CQRS Example                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  E-commerce Product Catalog:                                │
│                                                              │
│  Write Model (Command):                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Command: UpdateProductPrice                           │ │
│  │  - Validate business rules                             │ │
│  │  - Check authorization                                 │ │
│  │  - Update products table                              │ │
│  │  - Publish ProductPriceUpdated event                  │ │
│  │                                                         │ │
│  │  class Product {                                       │ │
│  │    id, name, description, price, inventory, ...       │ │
│  │  }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Read Model (Query):                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  View 1: ProductSearchView (for search page)          │ │
│  │  { id, name, price, thumbnail, rating }               │ │
│  │  Stored in: Elasticsearch                              │ │
│  │                                                         │ │
│  │  View 2: ProductDetailView (for product page)         │ │
│  │  { id, name, description, price, images, reviews }    │ │
│  │  Stored in: MongoDB                                    │ │
│  │                                                         │ │
│  │  View 3: AdminProductView (for admin panel)           │ │
│  │  { id, name, price, inventory, sales_data }          │ │
│  │  Stored in: PostgreSQL                                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Event Sourcing

```
┌─────────────────────────────────────────────────────────────┐
│                    Event Sourcing                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traditional: Store current state                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Account: { id: 123, balance: $150 }                   │ │
│  │                                                         │ │
│  │  How did we get here? Unknown!                        │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Event Sourcing: Store events that led to current state    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Event Log for Account 123:                            │ │
│  │  ┌─────┬────────────────────────┬────────────────────┐│ │
│  │  │ Seq │ Event                  │ Timestamp          ││ │
│  │  ├─────┼────────────────────────┼────────────────────┤│ │
│  │  │ 1   │ AccountOpened($0)      │ 2024-01-01 09:00  ││ │
│  │  │ 2   │ MoneyDeposited($200)   │ 2024-01-02 10:30  ││ │
│  │  │ 3   │ MoneyWithdrawn($50)    │ 2024-01-03 14:15  ││ │
│  │  └─────┴────────────────────────┴────────────────────┘│ │
│  │                                                         │ │
│  │  Current state = replay all events                      │ │
│  │  $0 + $200 - $50 = $150 ✓                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Key Principle: Events are immutable facts                 │
│  Never update or delete events—only append new ones        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why Event Sourcing?

```
┌─────────────────────────────────────────────────────────────┐
│                  Why Event Sourcing?                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Benefits:                                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✓ Complete Audit Trail                                 │ │
│  │   Every change is recorded with timestamp              │ │
│  │                                                         │ │
│  │ ✓ Temporal Queries                                     │ │
│  │   "What was the balance on Jan 15?"                   │ │
│  │   Replay events up to that date                       │ │
│  │                                                         │ │
│  │ ✓ Debugging                                            │ │
│  │   Replay events to reproduce bugs                     │ │
│  │                                                         │ │
│  │ ✓ Rebuild Views                                        │ │
│  │   Create new projections from event history           │ │
│  │                                                         │ │
│  │ ✓ Event-Driven Integration                            │ │
│  │   Events already exist for other services             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Challenges:                                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✗ Learning Curve                                       │ │
│  │   Different mental model                               │ │
│  │                                                         │ │
│  │ ✗ Event Schema Evolution                               │ │
│  │   Changing event structure is complex                  │ │
│  │                                                         │ │
│  │ ✗ Eventual Consistency                                 │ │
│  │   Projections may lag behind events                   │ │
│  │                                                         │ │
│  │ ✗ Storage Growth                                       │ │
│  │   Events accumulate over time                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Event Store

```
┌─────────────────────────────────────────────────────────────┐
│                      Event Store                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Structure:                                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Stream: Order-12345                                   │ │
│  │  ┌─────┬───────────────┬─────────┬──────────────────┐ │ │
│  │  │ Seq │ Event Type    │ Version │ Data             │ │ │
│  │  ├─────┼───────────────┼─────────┼──────────────────┤ │ │
│  │  │ 1   │ OrderCreated  │ 1       │ {items, total}   │ │ │
│  │  │ 2   │ ItemAdded     │ 1       │ {item, newTotal} │ │ │
│  │  │ 3   │ OrderSubmitted│ 1       │ {submittedAt}    │ │ │
│  │  └─────┴───────────────┴─────────┴──────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Key Operations:                                             │
│  • Append(streamId, events, expectedVersion)              │
│  • Read(streamId, fromVersion)                             │
│  • Subscribe(streamId, handler)                            │
│                                                              │
│  Optimistic Concurrency:                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Read events, current version = 5                   │ │
│  │  2. Make changes                                        │ │
│  │  3. Append(events, expectedVersion=5)                  │ │
│  │  4. If version != 5 → ConflictException               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Popular Event Stores:                                      │
│  • EventStoreDB (purpose-built)                            │
│  • PostgreSQL (with append-only table)                     │
│  • Kafka (as event log)                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## CQRS + Event Sourcing

```
┌─────────────────────────────────────────────────────────────┐
│               CQRS + Event Sourcing                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Combined Architecture:                                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                                                         ││
│  │  Commands              Event Store         Projections ││
│  │  ────────              ───────────         ─────────── ││
│  │                                                         ││
│  │  ┌────────┐         ┌─────────────┐      ┌──────────┐ ││
│  │  │Create  │         │ OrderCreated│      │ Order    │ ││
│  │  │Order   │──────▶  │ ItemAdded   │─────▶│ Summary  │ ││
│  │  └────────┘   │     │ OrderPaid   │  │   │ View     │ ││
│  │               │     └─────────────┘  │   └──────────┘ ││
│  │  ┌────────┐   │                      │                 ││
│  │  │Add     │───┘     Event Store      │   ┌──────────┐ ││
│  │  │Item    │         (Source of       └──▶│ Customer │ ││
│  │  └────────┘          Truth)              │ Orders   │ ││
│  │                                          │ View     │ ││
│  │  ┌────────┐                             └──────────┘ ││
│  │  │Pay     │──┐                                        ││
│  │  │Order   │  │                          ┌──────────┐ ││
│  │  └────────┘  └─────────────────────────▶│ Analytics│ ││
│  │                                          │ View     │ ││
│  │              Queries                    └──────────┘ ││
│  │              ───────                                   ││
│  │  ┌────────┐  ┌────────┐  ┌────────┐                  ││
│  │  │ Get    │  │ List   │  │ Get    │  Read from       ││
│  │  │ Order  │  │ Orders │  │ Stats  │  projections     ││
│  │  └────────┘  └────────┘  └────────┘                  ││
│  │                                                         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Projections

```
┌─────────────────────────────────────────────────────────────┐
│                      Projections                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Projection: Event handler that builds a read model         │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  class OrderSummaryProjection:                         │ │
│  │                                                         │ │
│  │    def handle(event):                                  │ │
│  │      match event:                                      │ │
│  │        case OrderCreated:                              │ │
│  │          insert({                                      │ │
│  │            id: event.orderId,                         │ │
│  │            status: "created",                         │ │
│  │            total: 0                                   │ │
│  │          })                                            │ │
│  │                                                         │ │
│  │        case ItemAdded:                                 │ │
│  │          update(event.orderId, {                      │ │
│  │            total: total + event.price                 │ │
│  │          })                                            │ │
│  │                                                         │ │
│  │        case OrderPaid:                                 │ │
│  │          update(event.orderId, {                      │ │
│  │            status: "paid",                            │ │
│  │            paidAt: event.timestamp                    │ │
│  │          })                                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Rebuilding Projections:                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Create new projection table                        │ │
│  │  2. Replay all events from event store                │ │
│  │  3. Switch read traffic to new table                  │ │
│  │  4. Delete old table                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Considerations

```
┌─────────────────────────────────────────────────────────────┐
│             Implementation Considerations                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Snapshots (Performance Optimization)                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Problem: 10,000 events to replay = slow              │ │
│  │                                                         │ │
│  │  Solution: Periodic snapshots                          │ │
│  │                                                         │ │
│  │  Events: [1..1000] → Snapshot at 1000 → [1001..1050]  │ │
│  │                                                         │ │
│  │  To get current state:                                 │ │
│  │  1. Load snapshot (state at event 1000)               │ │
│  │  2. Replay events 1001-1050 only                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Event Schema Evolution                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  V1: { orderId, amount }                               │ │
│  │  V2: { orderId, amount, currency }                    │ │
│  │                                                         │ │
│  │  Strategies:                                            │ │
│  │  • Upcasting: Transform old events when reading       │ │
│  │  • Versioned handlers: Handle multiple versions       │ │
│  │  • Copy-and-transform: Migrate to new stream          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Consistency                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Projections are eventually consistent!                │ │
│  │                                                         │ │
│  │  Command ──▶ Event Store ──▶ Projection                │ │
│  │                   ↓                ↓                    │ │
│  │              Immediate        Async update             │ │
│  │                                                         │ │
│  │  Handle with:                                          │ │
│  │  • Read-your-writes: Read from event store for writes│ │
│  │  • Polling: Wait for projection to catch up           │ │
│  │  • Accept eventual consistency in UI                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  4. When to Use                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Good fit:                                              │ │
│  │  • Complex domains with rich business logic           │ │
│  │  • Audit requirements                                  │ │
│  │  • Different read/write patterns                      │ │
│  │  • Event-driven integration                           │ │
│  │                                                         │ │
│  │  Not a good fit:                                       │ │
│  │  • Simple CRUD applications                           │ │
│  │  • Strong consistency requirements                    │ │
│  │  • Small teams unfamiliar with the patterns          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is CQRS? Why would you use it?
2. What is event sourcing?

### Intermediate
3. How do projections work in event sourcing?
4. How do you handle event schema changes?

### Advanced
5. Design an event-sourced order management system.
6. How would you handle the eventual consistency in CQRS?

---

[← Previous: Event-Driven Architecture](../02-event-driven/README.md) | [Next: Saga Pattern →](../04-saga-pattern/README.md)
