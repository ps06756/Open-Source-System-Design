# Design a Payment System

Design a payment processing system like Stripe or PayPal.

## 1. Requirements

### Functional Requirements
- Process payments (credit card, bank transfer)
- Refunds and chargebacks
- Multi-currency support
- Recurring payments (subscriptions)
- Fraud detection
- Ledger and reconciliation

### Non-Functional Requirements
- High availability (99.99%)
- Strong consistency (no double charges)
- PCI DSS compliance
- Audit trail for all transactions
- Scale: 1M transactions/day

### Capacity Estimation

```
Transactions: 1M/day ≈ 12 TPS average, 100 TPS peak
Transaction data: 1M × 1 KB = 1 GB/day
Ledger entries: 1M × 3 entries = 3M/day
Retention: Forever (regulatory)
```

## 2. High-Level Design

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ┌────────┐     ┌──────────────┐     ┌────────────────────┐   │
│  │Merchant│────▶│  API Gateway │────▶│  Payment Service   │   │
│  └────────┘     │  (Auth, TLS) │     └─────────┬──────────┘   │
│                 └──────────────┘               │              │
│                                                │              │
│    ┌───────────────────────────────────────────┼───────────┐ │
│    │                                           │           │ │
│    ▼                                           ▼           │ │
│ ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │ │
│ │    Fraud     │  │    Risk      │  │     Ledger       │   │ │
│ │   Service    │  │   Service    │  │    Service       │   │ │
│ └──────────────┘  └──────────────┘  └──────────────────┘   │ │
│                                                              │ │
│ ┌──────────────────────────────────────────────────────────┐│ │
│ │              Payment Processor Adapters                   ││ │
│ │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     ││ │
│ │  │  Visa   │  │MasterCard│  │  Bank   │  │ PayPal  │     ││ │
│ │  └─────────┘  └─────────┘  └─────────┘  └─────────┘     ││ │
│ └──────────────────────────────────────────────────────────┘│ │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 3. Payment Flow

```
┌────────────────────────────────────────────────────────────────┐
│                    Payment Flow                                │
│                                                                │
│  1. AUTHORIZE: Reserve funds                                  │
│     Merchant → Payment Service → Card Network → Bank          │
│     Response: authorization_code                              │
│                                                                │
│  2. CAPTURE: Actually transfer funds                          │
│     Merchant → Payment Service → Card Network → Bank          │
│     Funds move from customer to merchant's account            │
│                                                                │
│  Combined flow (auth + capture):                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  User ──▶ Merchant ──▶ Payment API ──▶ Fraud Check      │  │
│  │                                              │           │  │
│  │                                              ▼           │  │
│  │  Card Network ◀── PSP Adapter ◀── Risk Assessment      │  │
│  │        │                                                 │  │
│  │        ▼                                                 │  │
│  │  Issuing Bank (auth) ──▶ Response ──▶ Ledger Update    │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Settlement (end of day):                                     │
│  - Batch all captured transactions                           │
│  - Calculate net amounts                                      │
│  - Transfer to merchant bank accounts                        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Payment Service Implementation

```python
class PaymentService:
    def __init__(self):
        self.fraud_service = FraudService()
        self.risk_service = RiskService()
        self.ledger_service = LedgerService()
        self.psp_adapters = {
            'visa': VisaAdapter(),
            'mastercard': MasterCardAdapter(),
            'bank_transfer': BankTransferAdapter()
        }

    async def process_payment(self, payment_request):
        """Process a payment request"""
        payment_id = self.generate_idempotency_key(payment_request)

        # Check idempotency
        existing = await self.get_payment(payment_id)
        if existing:
            return existing

        # Create payment record
        payment = Payment(
            id=payment_id,
            amount=payment_request.amount,
            currency=payment_request.currency,
            merchant_id=payment_request.merchant_id,
            status='PENDING'
        )
        await self.save_payment(payment)

        try:
            # Fraud check
            fraud_result = await self.fraud_service.check(payment_request)
            if fraud_result.is_fraudulent:
                payment.status = 'DECLINED_FRAUD'
                await self.save_payment(payment)
                return payment

            # Risk assessment
            risk_result = await self.risk_service.assess(payment_request)
            if risk_result.requires_review:
                payment.status = 'PENDING_REVIEW'
                await self.save_payment(payment)
                return payment

            # Process with PSP
            psp = self.get_psp_adapter(payment_request.payment_method)
            result = await psp.authorize_and_capture(payment_request)

            if result.success:
                payment.status = 'COMPLETED'
                payment.processor_response = result.data

                # Update ledger
                await self.ledger_service.record_payment(payment)
            else:
                payment.status = 'DECLINED'
                payment.decline_reason = result.error_code

            await self.save_payment(payment)
            return payment

        except Exception as e:
            payment.status = 'ERROR'
            payment.error = str(e)
            await self.save_payment(payment)
            raise
```

## 4. Double-Entry Ledger

```
┌────────────────────────────────────────────────────────────────┐
│                Double-Entry Ledger                             │
│                                                                │
│  Every transaction has equal debits and credits               │
│                                                                │
│  Payment of $100 from Customer to Merchant:                   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Entry 1 (Debit):                                        │  │
│  │  Account: customer_payment_account                       │  │
│  │  Type: DEBIT                                              │  │
│  │  Amount: $100                                             │  │
│  │                                                          │  │
│  │  Entry 2 (Credit):                                       │  │
│  │  Account: merchant_receivable_account                    │  │
│  │  Type: CREDIT                                             │  │
│  │  Amount: $100                                             │  │
│  │                                                          │  │
│  │  Entry 3 (Fee - Debit from merchant):                   │  │
│  │  Account: merchant_receivable_account                    │  │
│  │  Type: DEBIT                                              │  │
│  │  Amount: $2.90 (2.9% fee)                                │  │
│  │                                                          │  │
│  │  Entry 4 (Fee - Credit to platform):                    │  │
│  │  Account: platform_revenue_account                       │  │
│  │  Type: CREDIT                                             │  │
│  │  Amount: $2.90                                            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Sum of DEBITs = Sum of CREDITs (always!)                    │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Ledger Implementation

```python
class LedgerService:
    async def record_payment(self, payment):
        """Record payment in ledger with double-entry"""
        entries = []

        # Debit customer
        entries.append(LedgerEntry(
            account_id=f"customer:{payment.customer_id}",
            payment_id=payment.id,
            type='DEBIT',
            amount=payment.amount,
            currency=payment.currency
        ))

        # Credit merchant (minus fees)
        fee = self.calculate_fee(payment)
        merchant_amount = payment.amount - fee

        entries.append(LedgerEntry(
            account_id=f"merchant:{payment.merchant_id}",
            payment_id=payment.id,
            type='CREDIT',
            amount=merchant_amount,
            currency=payment.currency
        ))

        # Credit platform fees
        entries.append(LedgerEntry(
            account_id="platform:fees",
            payment_id=payment.id,
            type='CREDIT',
            amount=fee,
            currency=payment.currency
        ))

        # Atomic insert (transaction)
        async with self.db.transaction():
            for entry in entries:
                await self.db.insert('ledger_entries', entry)

            # Validate double-entry
            total_debits = sum(e.amount for e in entries if e.type == 'DEBIT')
            total_credits = sum(e.amount for e in entries if e.type == 'CREDIT')

            if total_debits != total_credits:
                raise LedgerImbalanceError()
```

## 5. Idempotency

```
┌────────────────────────────────────────────────────────────────┐
│                   Idempotency                                  │
│                                                                │
│  Problem: Network failures can cause duplicate requests       │
│                                                                │
│  Solution: Idempotency keys                                   │
│  - Client sends unique key with each request                 │
│  - Server stores result keyed by idempotency key             │
│  - Repeat requests return cached result                       │
│                                                                │
│  Request:                                                      │
│  POST /payments                                                │
│  Idempotency-Key: ord_123_pay_001                             │
│                                                                │
│  Implementation:                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  CREATE TABLE idempotency_keys (                         │  │
│  │      key VARCHAR(64) PRIMARY KEY,                        │  │
│  │      response JSONB,                                      │  │
│  │      status VARCHAR(20),  -- pending, completed          │  │
│  │      created_at TIMESTAMP,                               │  │
│  │      expires_at TIMESTAMP                                │  │
│  │  );                                                       │  │
│  │                                                          │  │
│  │  async def process_with_idempotency(key, handler):       │  │
│  │      # Try to acquire lock                               │  │
│  │      existing = await db.get_or_create(key)              │  │
│  │      if existing.status == 'completed':                  │  │
│  │          return existing.response                        │  │
│  │      # Process and store result                          │  │
│  │      result = await handler()                            │  │
│  │      await db.update(key, response=result)               │  │
│  │      return result                                        │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 6. Fraud Detection

```python
class FraudService:
    def __init__(self):
        self.rules_engine = RulesEngine()
        self.ml_model = FraudMLModel()

    async def check(self, payment):
        """Check payment for fraud"""
        # Gather signals
        signals = await self.gather_signals(payment)

        # Rule-based checks
        rule_result = self.rules_engine.evaluate(signals)
        if rule_result.block:
            return FraudResult(is_fraudulent=True, reason=rule_result.reason)

        # ML model scoring
        fraud_score = self.ml_model.predict(signals)

        return FraudResult(
            is_fraudulent=fraud_score > 0.8,
            score=fraud_score,
            requires_review=0.5 < fraud_score < 0.8
        )

    async def gather_signals(self, payment):
        """Collect fraud detection signals"""
        return {
            # Velocity checks
            'transactions_last_hour': await self.count_recent_txns(
                payment.customer_id, hours=1
            ),
            'amount_last_24h': await self.sum_recent_amount(
                payment.customer_id, hours=24
            ),

            # Device/location
            'device_fingerprint': payment.device_fingerprint,
            'ip_country': self.geoip.lookup(payment.ip_address),
            'card_country': payment.card.country,

            # Behavioral
            'is_new_customer': payment.customer.created_days_ago < 7,
            'is_new_card': payment.card.added_days_ago < 1,
            'shipping_billing_match': self.addresses_match(
                payment.shipping, payment.billing
            ),

            # Historical
            'previous_chargebacks': payment.customer.chargeback_count,
            'account_age_days': payment.customer.created_days_ago
        }
```

## 7. Reconciliation

```
┌────────────────────────────────────────────────────────────────┐
│                   Reconciliation                               │
│                                                                │
│  Daily process to ensure our records match external systems   │
│                                                                │
│  1. Fetch settlement reports from PSPs                       │
│  2. Compare with our ledger entries                          │
│  3. Flag discrepancies for investigation                     │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Reconciliation Types:                                   │  │
│  │                                                          │  │
│  │  1. PSP Reconciliation                                   │  │
│  │     Our transactions vs PSP settlement report            │  │
│  │                                                          │  │
│  │  2. Bank Reconciliation                                  │  │
│  │     Expected deposits vs actual bank statement           │  │
│  │                                                          │  │
│  │  3. Internal Reconciliation                              │  │
│  │     Ledger debits = credits (should always balance)     │  │
│  │                                                          │  │
│  │  Discrepancy types:                                      │  │
│  │  - Missing: In our system, not in PSP                   │  │
│  │  - Extra: In PSP, not in our system                     │  │
│  │  - Amount mismatch                                       │  │
│  │  - Status mismatch                                       │  │
│  │                                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## 8. Database Schema

```sql
-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    merchant_id UUID NOT NULL,
    customer_id UUID,
    amount DECIMAL(19,4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    payment_method VARCHAR(50),
    processor VARCHAR(50),
    processor_transaction_id VARCHAR(100),
    idempotency_key VARCHAR(64) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP,
    INDEX idx_merchant (merchant_id, created_at),
    INDEX idx_customer (customer_id, created_at)
);

-- Ledger
CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    account_id VARCHAR(100) NOT NULL,
    payment_id UUID NOT NULL,
    type VARCHAR(10) NOT NULL,  -- DEBIT or CREDIT
    amount DECIMAL(19,4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    balance_after DECIMAL(19,4),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_account (account_id, created_at),
    INDEX idx_payment (payment_id)
);

-- Account balances (materialized view)
CREATE TABLE account_balances (
    account_id VARCHAR(100) PRIMARY KEY,
    balance DECIMAL(19,4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    last_updated TIMESTAMP
);
```

## 9. Key Takeaways

1. **Idempotency** - prevent double charges
2. **Double-entry ledger** - financial accuracy and auditability
3. **Async settlement** - authorize quick, settle in batch
4. **Fraud detection** - rules + ML hybrid approach
5. **Reconciliation** - daily verification with external systems
6. **PCI compliance** - never store raw card data

---

Next: [Hotel Booking](../08-hotel-booking/README.md)
