# Saga Pattern

The Saga pattern manages distributed transactions across multiple services by breaking them into a sequence of local transactions with compensating actions for rollback.

## Table of Contents
1. [Why Sagas?](#why-sagas)
2. [Saga Types](#saga-types)
3. [Compensation](#compensation)
4. [Implementation](#implementation)
5. [Interview Questions](#interview-questions)

---

## Why Sagas?

```
┌─────────────────────────────────────────────────────────────┐
│                       Why Sagas?                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Distributed transactions                          │
│                                                              │
│  Monolith (Easy):                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  BEGIN TRANSACTION                                      │ │
│  │    1. Create order                                      │ │
│  │    2. Reserve inventory                                 │ │
│  │    3. Charge payment                                    │ │
│  │    4. Schedule shipping                                 │ │
│  │  COMMIT  (or ROLLBACK if any step fails)               │ │
│  └────────────────────────────────────────────────────────┘ │
│  ACID guarantees from single database!                     │
│                                                              │
│  Microservices (Hard):                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │  Order   │ │Inventory │ │ Payment  │ │ Shipping │      │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │      │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘      │
│       │            │            │            │             │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐       │
│  │ DB      │  │ DB      │  │ DB      │  │ DB      │       │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │
│  Each service has its own database!                        │
│  Cannot use distributed transactions (2PC is slow/risky)   │
│                                                              │
│  Solution: Saga Pattern                                     │
│  • Sequence of local transactions                          │
│  • Each step has a compensating action                     │
│  • Eventually consistent                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Saga Types

### Choreography-Based Saga

```
┌─────────────────────────────────────────────────────────────┐
│               Choreography-Based Saga                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Each service listens for events and publishes next event  │
│                                                              │
│  Happy Path:                                                │
│  ┌───────┐   OrderCreated   ┌───────────┐                  │
│  │ Order │─────────────────▶│ Inventory │                  │
│  │Service│                  │  Service  │                  │
│  └───────┘                  └─────┬─────┘                  │
│                                   │ InventoryReserved       │
│                                   ▼                         │
│  ┌───────┐   PaymentCharged  ┌───────────┐                 │
│  │Shipping◀──────────────────│  Payment  │                 │
│  │Service│                   │  Service  │                 │
│  └───────┘                   └───────────┘                 │
│                                                              │
│  Failure Path (Payment fails):                             │
│  ┌───────┐   OrderCreated   ┌───────────┐                  │
│  │ Order │─────────────────▶│ Inventory │                  │
│  │Service│                  │  Service  │                  │
│  └───┬───┘                  └─────┬─────┘                  │
│      │                            │ InventoryReserved       │
│      │                            ▼                         │
│      │  PaymentFailed        ┌───────────┐                 │
│      │◀──────────────────────│  Payment  │                 │
│      │                       │  Service  │                 │
│      │                       └───────────┘                 │
│      │ OrderCancelled                                       │
│      └──────────────────────▶ Inventory releases stock     │
│                                                              │
│  ✓ Loose coupling                                          │
│  ✓ Simple for 2-3 step sagas                              │
│  ✗ Hard to understand full flow                           │
│  ✗ Cyclic dependencies possible                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Orchestration-Based Saga

```
┌─────────────────────────────────────────────────────────────┐
│               Orchestration-Based Saga                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Central orchestrator controls the saga                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                  Order Saga Orchestrator               │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  State Machine:                                   │ │ │
│  │  │  PENDING → INVENTORY_RESERVED → PAYMENT_CHARGED  │ │ │
│  │  │         → SHIPPING_SCHEDULED → COMPLETED         │ │ │
│  │  │                                                    │ │ │
│  │  │  On failure: transition to compensation states   │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│           │           │            │           │            │
│           ▼           ▼            ▼           ▼            │
│      ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │
│      │ Order  │  │Inventory│  │Payment │  │Shipping│        │
│      │Service │  │Service │  │Service │  │Service │        │
│      └────────┘  └────────┘  └────────┘  └────────┘        │
│                                                              │
│  Orchestrator implementation:                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  async def execute_order_saga(order):                  │ │
│  │    try:                                                 │ │
│  │      await inventory.reserve(order.items)              │ │
│  │      await payment.charge(order.total)                 │ │
│  │      await shipping.schedule(order)                    │ │
│  │      await order.complete()                            │ │
│  │    except InventoryError:                              │ │
│  │      await order.cancel()                              │ │
│  │    except PaymentError:                                │ │
│  │      await inventory.release(order.items)              │ │
│  │      await order.cancel()                              │ │
│  │    except ShippingError:                               │ │
│  │      await payment.refund(order.total)                 │ │
│  │      await inventory.release(order.items)              │ │
│  │      await order.cancel()                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ✓ Clear flow visibility                                   │
│  ✓ Easier to test and debug                               │
│  ✓ Better for complex multi-step sagas                    │
│  ✗ Orchestrator can be single point of failure            │
│  ✗ More coupling through orchestrator                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Compensation

```
┌─────────────────────────────────────────────────────────────┐
│                     Compensation                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Every step needs a compensating action (undo)             │
│                                                              │
│  ┌────────────────────┬────────────────────────────────┐   │
│  │ Action             │ Compensation                    │   │
│  ├────────────────────┼────────────────────────────────┤   │
│  │ Create Order       │ Cancel Order                    │   │
│  │ Reserve Inventory  │ Release Inventory               │   │
│  │ Charge Payment     │ Refund Payment                  │   │
│  │ Schedule Shipping  │ Cancel Shipping                 │   │
│  │ Send Email         │ Send Cancellation Email        │   │
│  └────────────────────┴────────────────────────────────┘   │
│                                                              │
│  Compensation order (reverse):                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Forward:  T1 → T2 → T3 → T4 (fail)                   │ │
│  │                                                         │ │
│  │  Compensate: C3 → C2 → C1                              │ │
│  │              ↑     ↑     ↑                              │ │
│  │           Undo  Undo  Undo                             │ │
│  │            T3    T2    T1                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Important: Compensation must be idempotent!               │
│  (Can be called multiple times safely)                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  # Bad: Non-idempotent                                  │ │
│  │  def refund():                                          │ │
│  │      account.balance += 100  # Called twice = +200!    │ │
│  │                                                         │ │
│  │  # Good: Idempotent                                     │ │
│  │  def refund(payment_id):                               │ │
│  │      if not refunds.exists(payment_id):                │ │
│  │          account.balance += 100                        │ │
│  │          refunds.create(payment_id)                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Semantic Rollback

```
┌─────────────────────────────────────────────────────────────┐
│                  Semantic Rollback                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Not all actions can be truly "undone"                     │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Scenario: Email was sent, saga fails later            │ │
│  │                                                         │ │
│  │  Can't "unsend" email!                                 │ │
│  │                                                         │ │
│  │  Compensation: Send another email                      │ │
│  │  "Your order has been cancelled. We apologize..."     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Scenario: External payment was processed              │ │
│  │                                                         │ │
│  │  Compensation: Issue refund (not instant!)             │ │
│  │  May take 3-5 business days                            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Design tip: Delay irreversible actions until late        │
│  in the saga when possible                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Saga State Machine

```
┌─────────────────────────────────────────────────────────────┐
│                   Saga State Machine                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Order Saga States:                                         │
│                                                              │
│  ┌─────────┐                                                │
│  │ PENDING │                                                │
│  └────┬────┘                                                │
│       │ reserveInventory()                                  │
│       ▼                                                      │
│  ┌─────────────────────┐      ┌─────────────────┐          │
│  │ INVENTORY_RESERVED  │─────▶│ INVENTORY_FAILED│          │
│  └─────────┬───────────┘      └────────┬────────┘          │
│            │ chargePayment()           │                    │
│            ▼                           ▼                    │
│  ┌─────────────────────┐      ┌─────────────────┐          │
│  │  PAYMENT_CHARGED    │─────▶│ PAYMENT_FAILED  │          │
│  └─────────┬───────────┘      └────────┬────────┘          │
│            │ scheduleShipping()        │ releaseInventory() │
│            ▼                           ▼                    │
│  ┌─────────────────────┐      ┌─────────────────┐          │
│  │SHIPPING_SCHEDULED   │─────▶│ SHIPPING_FAILED │          │
│  └─────────┬───────────┘      └────────┬────────┘          │
│            │                           │ refund(),         │
│            ▼                           │ releaseInventory()│
│  ┌─────────────────────┐      ┌────────▼────────┐          │
│  │     COMPLETED       │      │    CANCELLED    │          │
│  └─────────────────────┘      └─────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Example

```python
# Saga Orchestrator Implementation
class OrderSaga:
    def __init__(self, order_id):
        self.order_id = order_id
        self.state = "PENDING"
        self.completed_steps = []

    async def execute(self):
        try:
            # Step 1: Reserve Inventory
            await self._execute_step(
                action=inventory_service.reserve,
                compensation=inventory_service.release,
                next_state="INVENTORY_RESERVED"
            )

            # Step 2: Charge Payment
            await self._execute_step(
                action=payment_service.charge,
                compensation=payment_service.refund,
                next_state="PAYMENT_CHARGED"
            )

            # Step 3: Schedule Shipping
            await self._execute_step(
                action=shipping_service.schedule,
                compensation=shipping_service.cancel,
                next_state="SHIPPING_SCHEDULED"
            )

            self.state = "COMPLETED"

        except SagaStepError as e:
            await self._compensate()
            self.state = "CANCELLED"
            raise

    async def _execute_step(self, action, compensation, next_state):
        try:
            await action(self.order_id)
            self.completed_steps.append(compensation)
            self.state = next_state
        except Exception as e:
            raise SagaStepError(f"Failed at {next_state}")

    async def _compensate(self):
        # Execute compensations in reverse order
        for compensation in reversed(self.completed_steps):
            try:
                await compensation(self.order_id)
            except Exception as e:
                # Log and continue - compensation should be retried
                logger.error(f"Compensation failed: {e}")
```

### Saga Persistence

```
┌─────────────────────────────────────────────────────────────┐
│                   Saga Persistence                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Store saga state for recovery:                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Table: saga_instances                                  │ │
│  │  ┌─────────────┬────────────────────────────────────┐ │ │
│  │  │ saga_id     │ order-saga-12345                    │ │ │
│  │  │ type        │ ORDER_SAGA                          │ │ │
│  │  │ state       │ PAYMENT_CHARGED                     │ │ │
│  │  │ order_id    │ order-789                           │ │ │
│  │  │ steps       │ ["reserve", "charge"]               │ │ │
│  │  │ created_at  │ 2024-01-15T10:30:00Z               │ │ │
│  │  │ updated_at  │ 2024-01-15T10:30:05Z               │ │ │
│  │  └─────────────┴────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Recovery scenarios:                                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  1. Service restarts                                   │ │
│  │     → Load incomplete sagas, resume or compensate      │ │
│  │                                                         │ │
│  │  2. Timeout                                             │ │
│  │     → Mark saga as failed, trigger compensation        │ │
│  │                                                         │ │
│  │  3. Compensation failure                               │ │
│  │     → Retry with backoff, alert if persistent         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Best Practices

```
┌─────────────────────────────────────────────────────────────┐
│                    Best Practices                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Make steps idempotent                                  │
│     Both forward and compensation actions                  │
│                                                              │
│  2. Use correlation IDs                                     │
│     Track all saga events with same ID                     │
│                                                              │
│  3. Order steps carefully                                   │
│     Reversible steps first, irreversible last             │
│                                                              │
│  4. Handle partial failures                                │
│     What if compensation fails? Retry + alert              │
│                                                              │
│  5. Set timeouts                                            │
│     Don't wait forever for step completion                 │
│                                                              │
│  6. Consider semantic locking                              │
│     Prevent concurrent modifications during saga           │
│     e.g., Order status = "PROCESSING"                     │
│                                                              │
│  7. Use saga frameworks                                     │
│     • Temporal (recommended)                               │
│     • Camunda                                               │
│     • AWS Step Functions                                   │
│     • Axon Framework                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the Saga pattern?
2. Why can't we use traditional transactions in microservices?

### Intermediate
3. Compare choreography vs orchestration sagas.
4. What is compensation and why is it important?

### Advanced
5. Design a saga for an e-commerce checkout flow.
6. How do you handle compensation failures?

---

[← Previous: CQRS & Event Sourcing](../03-cqrs-event-sourcing/README.md) | [Next: Circuit Breaker →](../05-circuit-breaker/README.md)
