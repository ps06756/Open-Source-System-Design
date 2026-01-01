# Saga Pattern

The Saga pattern manages distributed transactions across multiple services without using traditional ACID transactions.

## Table of Contents
- [The Problem](#the-problem)
- [What is a Saga?](#what-is-a-saga)
- [Choreography vs Orchestration](#choreography-vs-orchestration)
- [Compensation](#compensation)
- [Implementation](#implementation)
- [Key Takeaways](#key-takeaways)

## The Problem

In microservices, operations span multiple services with separate databases.

```
┌────────────────────────────────────────────────────────┐
│           Distributed Transaction Problem              │
│                                                        │
│  Place Order:                                          │
│  1. Create order (Order Service)                      │
│  2. Reserve inventory (Inventory Service)             │
│  3. Charge payment (Payment Service)                  │
│  4. Send confirmation (Notification Service)          │
│                                                        │
│  What if step 3 fails?                                │
│  - Order is created ✓                                 │
│  - Inventory is reserved ✓                            │
│  - Payment failed ✗                                   │
│  - Notification never sent                            │
│                                                        │
│  System is in inconsistent state!                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Why Not 2PC?

Two-Phase Commit (2PC) has problems in microservices:
- Requires all participants to be available
- Locks resources during transaction
- Doesn't work across different databases
- Single coordinator is SPOF

## What is a Saga?

A Saga is a sequence of local transactions. Each step either succeeds and triggers the next, or fails and triggers compensating transactions.

```
┌────────────────────────────────────────────────────────┐
│                      Saga                              │
│                                                        │
│  Happy Path:                                           │
│  T1 ──▶ T2 ──▶ T3 ──▶ T4 ──▶ Success                 │
│                                                        │
│  Failure at T3:                                        │
│  T1 ──▶ T2 ──▶ T3(fail) ──▶ C2 ──▶ C1                │
│                    │                                   │
│              Compensation                              │
│              (undo T2, T1)                             │
│                                                        │
│  T = Transaction                                       │
│  C = Compensation                                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Choreography vs Orchestration

### Choreography

Services coordinate through events. No central controller.

```
┌────────────────────────────────────────────────────────┐
│                  Choreography                          │
│                                                        │
│  ┌─────────┐   OrderCreated   ┌───────────┐           │
│  │  Order  │─────────────────▶│ Inventory │           │
│  │ Service │                  │  Service  │           │
│  └─────────┘                  └─────┬─────┘           │
│       ▲                             │                  │
│       │                    InventoryReserved          │
│       │                             │                  │
│       │                             ▼                  │
│       │                       ┌───────────┐           │
│       │                       │  Payment  │           │
│       │                       │  Service  │           │
│       │                       └─────┬─────┘           │
│       │                             │                  │
│       │                      PaymentCharged           │
│       │                             │                  │
│       │                             ▼                  │
│       │   OrderCompleted      ┌───────────┐           │
│       └───────────────────────│Notification│          │
│                               │  Service  │           │
│                               └───────────┘           │
│                                                        │
│  Each service listens for events and acts             │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:**
- Loosely coupled
- Simple for small flows
- No single point of failure

**Cons:**
- Hard to understand complete flow
- Difficult to debug
- Cyclic dependencies possible

### Orchestration

Central coordinator manages the saga.

```
┌────────────────────────────────────────────────────────┐
│                   Orchestration                        │
│                                                        │
│                 ┌─────────────────┐                   │
│                 │   Orchestrator  │                   │
│                 │  (Order Saga)   │                   │
│                 └────────┬────────┘                   │
│                          │                             │
│         ┌────────────────┼────────────────┐           │
│         │                │                │           │
│         ▼                ▼                ▼           │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐     │
│  │  Order    │    │ Inventory │    │  Payment  │     │
│  │  Service  │    │  Service  │    │  Service  │     │
│  └───────────┘    └───────────┘    └───────────┘     │
│                                                        │
│  Orchestrator:                                         │
│  1. Call OrderService.create()                        │
│  2. Call InventoryService.reserve()                   │
│  3. Call PaymentService.charge()                      │
│  4. If any fails, call compensations                  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Pros:**
- Clear flow in one place
- Easier to understand and debug
- No cyclic dependencies

**Cons:**
- Central point of failure
- Orchestrator can become complex
- More coupling

### Comparison

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| Coupling | Loose | Tighter |
| Visibility | Distributed | Centralized |
| Complexity | In events | In orchestrator |
| Debugging | Harder | Easier |
| Best for | Simple flows | Complex flows |

## Compensation

Compensating transactions undo the effects of previous steps.

```
┌────────────────────────────────────────────────────────┐
│              Compensation Examples                     │
│                                                        │
│  Step              │ Compensation                      │
│  ──────────────────┼────────────────────────────────  │
│  Create Order      │ Cancel Order                     │
│  Reserve Inventory │ Release Inventory                │
│  Charge Payment    │ Refund Payment                   │
│  Send Email        │ Send Cancellation Email          │
│  Create Shipment   │ Cancel Shipment                  │
│                                                        │
│  Note: Some actions can't be compensated!             │
│  - Email sent (can only send correction)              │
│  - Physical item shipped                              │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Semantic Compensation

When exact undo isn't possible:

```
Original: Send welcome email
Compensation: Can't unsend!

Options:
1. Send correction email
2. Mark user for manual follow-up
3. Accept as non-critical

Original: Deduct loyalty points
Compensation: Add points back (may not be exact if expired)
```

## Implementation

### Orchestrator-Based Saga

```python
class OrderSaga:
    def __init__(self, services):
        self.order_service = services.order
        self.inventory_service = services.inventory
        self.payment_service = services.payment

    def execute(self, order_request):
        saga_state = SagaState()

        try:
            # Step 1: Create order
            order = self.order_service.create(order_request)
            saga_state.order_id = order.id

            # Step 2: Reserve inventory
            reservation = self.inventory_service.reserve(
                order.items
            )
            saga_state.reservation_id = reservation.id

            # Step 3: Charge payment
            payment = self.payment_service.charge(
                order.customer_id,
                order.total
            )
            saga_state.payment_id = payment.id

            # Success!
            self.order_service.complete(order.id)
            return order

        except Exception as e:
            # Compensate in reverse order
            self.compensate(saga_state)
            raise SagaFailedException(e)

    def compensate(self, state):
        if state.payment_id:
            self.payment_service.refund(state.payment_id)

        if state.reservation_id:
            self.inventory_service.release(state.reservation_id)

        if state.order_id:
            self.order_service.cancel(state.order_id)
```

### State Machine Approach

```python
class OrderSagaStateMachine:
    states = ['PENDING', 'ORDER_CREATED', 'INVENTORY_RESERVED',
              'PAYMENT_CHARGED', 'COMPLETED', 'COMPENSATING', 'FAILED']

    def __init__(self):
        self.state = 'PENDING'

    def handle(self, event):
        transitions = {
            ('PENDING', 'CreateOrder'): self.create_order,
            ('ORDER_CREATED', 'ReserveInventory'): self.reserve_inventory,
            ('INVENTORY_RESERVED', 'ChargePayment'): self.charge_payment,
            ('PAYMENT_CHARGED', 'Complete'): self.complete,
            # Compensation transitions
            ('PAYMENT_CHARGED', 'Fail'): self.start_compensation,
            ('COMPENSATING', 'RefundComplete'): self.release_inventory,
            ('COMPENSATING', 'ReleaseComplete'): self.cancel_order,
        }

        handler = transitions.get((self.state, event.type))
        if handler:
            handler(event)
```

### Saga with Message Queue

```
┌────────────────────────────────────────────────────────┐
│            Saga with Message Queue                     │
│                                                        │
│  ┌─────────────┐                                      │
│  │Orchestrator │                                      │
│  │             │──┐                                   │
│  └─────────────┘  │                                   │
│        ▲          │                                   │
│        │          ▼                                   │
│        │    ┌──────────────┐                         │
│        │    │    Kafka     │                         │
│        │    │              │                         │
│        │    │ saga.commands│──────┬─────────┐        │
│        │    │ saga.replies │      │         │        │
│        │    └──────────────┘      ▼         ▼        │
│        │          ▲         ┌─────────┐ ┌─────────┐  │
│        │          │         │ Order   │ │Inventory│  │
│        └──────────┴─────────│ Service │ │ Service │  │
│                             └─────────┘ └─────────┘  │
│                                                       │
│  Benefits:                                            │
│  - Durable messages                                   │
│  - Retry on failure                                   │
│  - Audit trail                                        │
│                                                       │
└────────────────────────────────────────────────────────┘
```

## Saga Considerations

### Idempotency

Each step must be idempotent (safe to retry).

```python
def reserve_inventory(order_id, items):
    # Check if already reserved
    existing = db.find_reservation(order_id)
    if existing:
        return existing  # Idempotent!

    # Create new reservation
    reservation = create_reservation(items)
    db.save(order_id, reservation)
    return reservation
```

### Timeout Handling

```
┌────────────────────────────────────────────────────────┐
│              Timeout Handling                          │
│                                                        │
│  Saga Step Timeout:                                    │
│  - Service doesn't respond in time                    │
│  - Options:                                            │
│    1. Retry with backoff                              │
│    2. Compensate and fail                             │
│    3. Mark for manual intervention                    │
│                                                        │
│  Saga Overall Timeout:                                 │
│  - Saga taking too long                               │
│  - Force compensation after X minutes                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Observability

```python
class SagaLogger:
    def log_step(self, saga_id, step, status, data):
        log = {
            "saga_id": saga_id,
            "step": step,
            "status": status,  # STARTED, COMPLETED, FAILED
            "data": data,
            "timestamp": datetime.now()
        }
        self.store.append(log)

# Enables:
# - Debugging failed sagas
# - Monitoring saga durations
# - Identifying bottlenecks
```

## Key Takeaways

1. **Sagas manage distributed transactions** without 2PC
2. **Choreography for simple**, orchestration for complex flows
3. **Design compensations** for every step
4. **Make steps idempotent** for safe retries
5. **Handle timeouts** explicitly
6. **Accept eventual consistency** - saga takes time

## Practice Questions

1. Design a saga for an e-commerce checkout flow.
2. How do you handle a compensation that fails?
3. When would you choose choreography over orchestration?
4. How do you test a saga implementation?

## Further Reading

- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Implementing Sagas](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga)
- [Life Beyond Distributed Transactions](https://queue.acm.org/detail.cfm?id=3025012)

---

Next: [Circuit Breaker](../05-circuit-breaker/README.md)
