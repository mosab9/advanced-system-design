# Design a Payment System

Financial transaction processing with high reliability.

---

## 1. Requirements

### Functional
- Process payments (credit card, debit, wallets)
- Handle refunds
- Transaction history
- Multiple currencies
- Fraud detection

### Non-Functional
- High consistency (no double charges)
- High availability
- PCI DSS compliance
- Audit trail
- Exactly-once processing

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   ┌─────────────┐     ┌─────────────────────────────────────────┐  │
│   │   Client    │────▶│           API Gateway                   │  │
│   └─────────────┘     │    (Auth, Rate Limiting, TLS)          │  │
│                       └──────────────────┬──────────────────────┘  │
│                                          │                          │
│                                          ▼                          │
│                       ┌─────────────────────────────────────────┐  │
│                       │         Payment Service                  │  │
│                       │   (Orchestration, Idempotency)          │  │
│                       └──────────────────┬──────────────────────┘  │
│                                          │                          │
│         ┌────────────────────────────────┼─────────────────────┐   │
│         │                                │                     │   │
│         ▼                                ▼                     ▼   │
│   ┌───────────┐               ┌───────────────┐       ┌─────────┐ │
│   │  Fraud    │               │    Ledger     │       │  PSP    │ │
│   │ Detection │               │   Service     │       │Connector│ │
│   └───────────┘               │ (Double-entry)│       └────┬────┘ │
│                               └───────────────┘            │      │
│                                                            ▼      │
│                                                  ┌────────────────┐│
│                                                  │ Stripe/Adyen/  ││
│                                                  │ PayPal         ││
│                                                  └────────────────┘│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Payment Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Payment Processing Flow                         │
│                                                                     │
│   1. Client initiates payment with idempotency_key                 │
│                                                                     │
│   2. Payment Service:                                              │
│      a. Check idempotency (return cached result if exists)        │
│      b. Validate request                                           │
│      c. Create PENDING transaction                                │
│      d. Call Fraud Detection                                       │
│                                                                     │
│   3. If fraud check passes:                                        │
│      a. Call PSP (Stripe) to authorize                            │
│      b. If authorized → Capture payment                           │
│      c. Record in Ledger                                           │
│      d. Update transaction to COMPLETED                           │
│                                                                     │
│   4. Return result (cached for idempotency)                       │
│                                                                     │
│   States: PENDING → AUTHORIZED → CAPTURED → COMPLETED             │
│                   → DECLINED                                       │
│                   → FAILED                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Idempotency

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Idempotency Implementation                      │
│                                                                     │
│   Problem: Network failure → Retry → Double charge                 │
│                                                                     │
│   Solution:                                                        │
│   - Client provides unique idempotency_key                         │
│   - Server stores: key → (status, response)                       │
│                                                                     │
│   Request 1: POST /pay {idempotency_key: "abc123", amount: 100}   │
│   → Process payment → Store result                                │
│   → Return {status: "success", txn_id: "xyz"}                     │
│                                                                     │
│   Request 2 (retry): POST /pay {idempotency_key: "abc123", ...}   │
│   → Found cached result                                           │
│   → Return {status: "success", txn_id: "xyz"} (same response)     │
│                                                                     │
│   Storage (Redis with TTL):                                        │
│   SETEX idempotency:abc123 86400 '{"status":"success",...}'       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5. Double-Entry Ledger

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Double-Entry Bookkeeping                        │
│                                                                     │
│   Every transaction has two entries:                               │
│   - Debit (money out)                                              │
│   - Credit (money in)                                              │
│   - Sum of all debits = Sum of all credits                        │
│                                                                     │
│   Example: User pays $100 for order                                │
│                                                                     │
│   ┌──────────────┬──────────────┬──────────────┬──────────────┐   │
│   │  Account     │   Debit      │   Credit     │   Balance    │   │
│   ├──────────────┼──────────────┼──────────────┼──────────────┤   │
│   │ User Wallet  │   $100       │              │   -$100      │   │
│   │ Merchant     │              │   $97        │   +$97       │   │
│   │ Platform Fee │              │   $3         │   +$3        │   │
│   └──────────────┴──────────────┴──────────────┴──────────────┘   │
│                                                                     │
│   Entries table:                                                   │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │ txn_id │ account_id │ amount │ type   │ created_at         │  │
│   │ T1     │ user_123   │ 100    │ DEBIT  │ 2024-01-15 10:00  │  │
│   │ T1     │ merchant_1 │ 97     │ CREDIT │ 2024-01-15 10:00  │  │
│   │ T1     │ platform   │ 3      │ CREDIT │ 2024-01-15 10:00  │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Handling Failures

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Failure Scenarios                               │
│                                                                     │
│   1. PSP timeout:                                                  │
│      - Don't know if payment succeeded                            │
│      - Query PSP for transaction status                           │
│      - Use reconciliation job to sync                             │
│                                                                     │
│   2. Service crash after PSP success:                              │
│      - Transaction in PSP, not in our DB                         │
│      - Reconciliation detects mismatch                            │
│      - Create missing ledger entries                              │
│                                                                     │
│   3. Partial failure (multi-step):                                 │
│      - Use Saga pattern with compensating transactions            │
│      - Example: Charge succeeded, inventory update failed         │
│        → Trigger refund                                            │
│                                                                     │
│   Reconciliation Job (daily):                                      │
│   - Fetch transactions from PSP                                   │
│   - Compare with internal ledger                                  │
│   - Flag discrepancies for review                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Data Model

```sql
-- Transactions
CREATE TABLE transactions (
    id              UUID PRIMARY KEY,
    idempotency_key VARCHAR(100) UNIQUE,
    user_id         UUID,
    amount          DECIMAL(19, 4),
    currency        VARCHAR(3),
    status          VARCHAR(20),
    psp_reference   VARCHAR(100),
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

-- Ledger entries
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY,
    transaction_id  UUID,
    account_id      UUID,
    amount          DECIMAL(19, 4),
    entry_type      VARCHAR(10),  -- DEBIT or CREDIT
    created_at      TIMESTAMP,
    INDEX (transaction_id),
    INDEX (account_id, created_at)
);
```

---

## 8. Interview Tips

**Key Points:**
- Idempotency for exactly-once processing
- Double-entry ledger for financial accuracy
- PSP integration patterns
- Failure handling and reconciliation
- PCI compliance considerations

**Questions to Ask:**
- Which payment methods?
- Multi-currency support?
- Recurring payments?
- Dispute/chargeback handling?
