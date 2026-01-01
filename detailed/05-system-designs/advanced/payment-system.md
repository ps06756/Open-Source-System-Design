# Design a Payment System

## 1. Requirements

### Functional Requirements
- Process payments (credit card, debit, bank transfer)
- Handle refunds
- Payment status tracking
- Transaction history
- Multi-currency support
- Fraud detection

### Non-Functional Requirements
- High reliability (never lose transactions)
- Exactly-once processing
- PCI-DSS compliance
- Low latency (< 1 second)
- High availability

### Scale Estimates
- 1M transactions per day
- Average transaction: $50
- Peak: 100 transactions per second

---

## 2. High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                  High-Level Architecture                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Client                               │ │
│  │            (Web/Mobile App)                            │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│                           ▼                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                 Payment Gateway                         │ │
│  │        (Public API, authentication)                    │ │
│  └────────────────────────┬───────────────────────────────┘ │
│                           │                                  │
│                           ▼                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Payment Service                            │ │
│  │     (Orchestrate payment flow)                         │ │
│  └─────────┬──────────────┬───────────────┬───────────────┘ │
│            │              │               │                  │
│            ▼              ▼               ▼                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│  │    Fraud     │ │   Ledger    │ │   Payment Service    │ │
│  │   Service    │ │   Service   │ │   Provider (PSP)     │ │
│  └──────────────┘ └──────────────┘ │ (Stripe, Adyen)      │ │
│                                     └──────────────────────┘ │
│                           │                                  │
│                           ▼                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Card Networks / Banks                      │ │
│  │           (Visa, Mastercard, Banks)                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Payment Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     Payment Flow                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Customer initiates payment                         │ │
│  │     ┌────────┐   POST /pay    ┌──────────────┐        │ │
│  │     │ Client │───────────────▶│ Payment      │        │ │
│  │     └────────┘  {amount,      │ Gateway      │        │ │
│  │                  card_token}  └──────────────┘        │ │
│  │                                                         │ │
│  │  2. Create payment record (PENDING)                    │ │
│  │     ┌──────────────┐   ┌──────────────────────┐       │ │
│  │     │ Payment      │──▶│ Database             │       │ │
│  │     │ Service      │   │ (payment_id, PENDING)│       │ │
│  │     └──────────────┘   └──────────────────────┘       │ │
│  │                                                         │ │
│  │  3. Fraud check                                        │ │
│  │     ┌──────────────┐   ┌──────────────┐               │ │
│  │     │ Fraud        │──▶│ Risk Score   │               │ │
│  │     │ Service      │   │ (approve/    │               │ │
│  │     └──────────────┘   │  decline)    │               │ │
│  │                        └──────────────┘               │ │
│  │                                                         │ │
│  │  4. Call Payment Service Provider                     │ │
│  │     ┌──────────────┐   ┌──────────────┐               │ │
│  │     │ Payment      │──▶│ PSP (Stripe) │               │ │
│  │     │ Service      │   │              │               │ │
│  │     └──────────────┘   └──────────────┘               │ │
│  │                                 │                      │ │
│  │  5. PSP processes with card network                   │ │
│  │                                 ▼                      │ │
│  │                        ┌──────────────┐               │ │
│  │                        │ Visa/MC      │               │ │
│  │                        │ → Issuer Bank│               │ │
│  │                        └──────────────┘               │ │
│  │                                 │                      │ │
│  │  6. Update status (SUCCESS/FAILED)                    │ │
│  │                                 ▼                      │ │
│  │     ┌──────────────┐   ┌──────────────┐               │ │
│  │     │ Ledger       │──▶│ Record in    │               │ │
│  │     │ Service      │   │ double-entry │               │ │
│  │     └──────────────┘   └──────────────┘               │ │
│  │                                                         │ │
│  │  7. Return result to client                           │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Idempotency

```
┌─────────────────────────────────────────────────────────────┐
│                      Idempotency                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Network failure after payment processed          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Client ──▶ Server ──▶ PSP (SUCCESS)                   │ │
│  │                                                         │ │
│  │  Server ──▶ Client: [TIMEOUT/ERROR]                    │ │
│  │                                                         │ │
│  │  Client retries ──▶ Double charge!                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Solution: Idempotency Key                                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Request:                                               │ │
│  │  POST /payments                                        │ │
│  │  Idempotency-Key: uuid-12345                          │ │
│  │  { amount: 100, card_token: "tok_xxx" }               │ │
│  │                                                         │ │
│  │  Server flow:                                           │ │
│  │  1. Check if idempotency_key exists in DB             │ │
│  │  2. If exists → return cached response                │ │
│  │  3. If not → process, store result with key           │ │
│  │                                                         │ │
│  │  Same request = same result (no double processing)    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Implementation:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Table: idempotency_keys                               │ │
│  │  ┌────────────────┬──────────────────────────────────┐│ │
│  │  │ key (PK)       │ uuid-12345                        ││ │
│  │  │ request_hash   │ SHA256(request_body)              ││ │
│  │  │ response       │ { status: success, ... }          ││ │
│  │  │ created_at     │ 2024-01-15 10:00:00              ││ │
│  │  │ expires_at     │ 2024-01-15 10:30:00              ││ │
│  │  └────────────────┴──────────────────────────────────┘│ │
│  │                                                         │ │
│  │  • TTL: 30 minutes (prevent stale data)               │ │
│  │  • Verify request_hash matches (prevent misuse)       │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Double-Entry Ledger

```
┌─────────────────────────────────────────────────────────────┐
│                  Double-Entry Ledger                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Principle: Every transaction has debit and credit         │
│  Total debits = Total credits (always balanced)            │
│                                                              │
│  Payment Example:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Customer pays $100 to Merchant                        │ │
│  │                                                         │ │
│  │  ┌──────────────┬────────┬────────┬─────────────────┐ │ │
│  │  │ Account      │ Debit  │ Credit │ Description     │ │ │
│  │  ├──────────────┼────────┼────────┼─────────────────┤ │ │
│  │  │ Customer     │ $100   │        │ Payment out     │ │ │
│  │  │ Merchant     │        │ $97    │ Payment in      │ │ │
│  │  │ Platform Fee │        │ $3     │ Commission      │ │ │
│  │  └──────────────┴────────┴────────┴─────────────────┘ │ │
│  │                                                         │ │
│  │  Total: Debit $100 = Credit $100 ✓                    │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Ledger Table:                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ entry_id       │ BIGINT (PK)                           │ │
│  │ transaction_id │ UUID (FK)                             │ │
│  │ account_id     │ BIGINT                                │ │
│  │ amount         │ DECIMAL(19,4)                         │ │
│  │ type           │ ENUM(debit, credit)                   │ │
│  │ currency       │ CHAR(3)                               │ │
│  │ created_at     │ TIMESTAMP                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Properties:                                                │
│  • Append-only (immutable)                                 │
│  • Never update/delete entries                             │
│  • Corrections via new reversal entries                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Reconciliation

```
┌─────────────────────────────────────────────────────────────┐
│                    Reconciliation                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Why reconcile?                                             │
│  • Our records vs PSP records may differ                   │
│  • Network failures, race conditions                       │
│  • Fraud, disputes                                          │
│                                                              │
│  Reconciliation Flow:                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Daily: Get settlement file from PSP               │ │
│  │     ┌──────────────────────────────────────────────┐  │ │
│  │     │ PSP Settlement File (T+1)                     │  │ │
│  │     │ txn_id, amount, status, timestamp            │  │ │
│  │     └──────────────────────────────────────────────┘  │ │
│  │                                                         │ │
│  │  2. Compare with our records                           │ │
│  │     ┌────────────────┬─────────────┬───────────────┐  │ │
│  │     │ Our Records    │ PSP Records │ Status        │  │ │
│  │     ├────────────────┼─────────────┼───────────────┤  │ │
│  │     │ txn_001: $100  │ $100        │ ✓ Match      │  │ │
│  │     │ txn_002: $50   │ $50         │ ✓ Match      │  │ │
│  │     │ txn_003: $75   │ (missing)   │ ✗ Mismatch  │  │ │
│  │     │ (missing)      │ txn_004:$30 │ ✗ Mismatch  │  │ │
│  │     └────────────────┴─────────────┴───────────────┘  │ │
│  │                                                         │ │
│  │  3. Handle mismatches                                  │ │
│  │     • Create reconciliation tasks                     │ │
│  │     • Manual review for discrepancies                 │ │
│  │     • Auto-correct known patterns                     │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Metrics:                                                   │
│  • Reconciliation rate: 99.9%+ expected                    │
│  • Mismatch rate: < 0.1%                                   │
│  • Time to resolve: < 24 hours                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Handling Failures

```
┌─────────────────────────────────────────────────────────────┐
│                  Handling Failures                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  State Machine for Payments:                                │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  CREATED ──▶ PROCESSING ──▶ SUCCESS                    │ │
│  │      │           │                                      │ │
│  │      │           └──▶ FAILED                           │ │
│  │      │                   │                              │ │
│  │      └──▶ CANCELLED      └──▶ REFUND_PENDING           │ │
│  │                                    │                    │ │
│  │                                    └──▶ REFUNDED        │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Timeout Handling:                                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  1. Payment stuck in PROCESSING for > 30 seconds      │ │
│  │  2. Background job queries PSP for status             │ │
│  │  3. If SUCCESS → update our record                    │ │
│  │  4. If FAILED → mark failed, notify customer          │ │
│  │  5. If UNKNOWN → flag for manual review               │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Retry Strategy:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  Retryable errors:                                     │ │
│  │  • Network timeout                                     │ │
│  │  • 5xx from PSP                                        │ │
│  │  • Rate limited                                        │ │
│  │                                                         │ │
│  │  Non-retryable errors:                                 │ │
│  │  • Card declined                                       │ │
│  │  • Insufficient funds                                  │ │
│  │  • Invalid card                                        │ │
│  │                                                         │ │
│  │  Retry with exponential backoff:                       │ │
│  │  1s → 2s → 4s → 8s → give up                         │ │
│  │                                                         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Summary

| Component | Choice | Reason |
|-----------|--------|--------|
| **Database** | PostgreSQL | ACID, transactions |
| **Idempotency** | Request keys + storage | Exactly-once |
| **Ledger** | Double-entry, append-only | Auditability |
| **PSP Integration** | Async with webhooks | Reliability |
| **Reconciliation** | Daily batch + real-time | Accuracy |

---

[Back to System Designs](../README.md)
