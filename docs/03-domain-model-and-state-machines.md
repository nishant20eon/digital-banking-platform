# 03 – Domain Model and State Machines

## 1. Core domain concepts

- **Customer:** Person or organization using the platform.
- **Account:** Financial account owned by a customer.
- **Available balance:** Funds eligible for new spending after active holds/restrictions.
- **Ledger balance:** Posted balance according to the ledger model.
- **Hold:** Reservation of funds for a transfer.
- **Beneficiary:** Saved recipient information.
- **Transfer:** Business request to move funds.
- **Saga:** Durable orchestration state for a multi-service transfer.
- **Ledger transaction:** Immutable accounting journal.
- **Ledger entry:** One debit or credit line in a journal.
- **Transaction projection:** Customer-facing read model of financial activity.
- **Outbox event:** Event persisted in the same local transaction as business state.
- **Inbox/processed event:** Record that a consumer has handled an event.

## 2. Transfer state machine

```text
RECEIVED
  ├── validation fails ──> REJECTED
  └── validation passes -> VALIDATING
                            |
                            v
                       FUNDS_RESERVING
                         ├── failure -> FAILED
                         └── success -> FUNDS_RESERVED
                                          |
                                          v
                                     LEDGER_POSTING
                                       ├── confirmed -> COMPLETED
                                       ├── definite failure -> COMPENSATING
                                       └── unknown outcome -> MANUAL_REVIEW / retry-query
                                                               |
                                                               v
                                                         COMPENSATING
                                                           ├── success -> FAILED
                                                           └── failure -> MANUAL_REVIEW
```

The exact transition sequence may be refined during implementation. Enforce transitions in domain/service logic; do not allow arbitrary status updates.

## 3. Account states

Suggested states:
- `PENDING_ACTIVATION`
- `ACTIVE`
- `FROZEN`
- `CLOSURE_PENDING`
- `CLOSED`

Define permitted transitions and who may invoke them. A closed account is not reactivated through a normal customer API.

## 4. Hold states

- `REQUESTED`
- `ACTIVE`
- `CONSUMED`
- `RELEASE_REQUESTED`
- `RELEASED`
- `EXPIRED`
- `MANUAL_REVIEW`

A hold must be uniquely associated with a transfer. Duplicate reserve/release commands must be safe.

## 5. Ledger posting states

- `RECEIVED`
- `POSTED`
- `REJECTED`

A posting reference is unique. A retry with the same reference and same payload returns the existing outcome. Same reference with a different payload is a conflict and must be investigated.

## 6. Reversal

A reversal is a new journal transaction referencing the original journal. Never mutate or delete original posted entries. Prevent duplicate reversal with a unique reversal reference and a rule limiting permitted reversal count/amount.

## 7. Domain invariants

1. A transfer amount is positive and uses a supported currency/scale.
2. A customer can only debit an account they own.
3. An account cannot reserve more than its available funds.
4. A transfer cannot complete without confirmed ledger posting.
5. A journal balances per currency.
6. A posted journal is immutable.
7. One business posting reference has at most one successful posting.
8. Duplicate idempotency key + same request maps to the original operation.
9. Duplicate key + different request is rejected.
10. A compensation is itself durable, retryable and auditable.
