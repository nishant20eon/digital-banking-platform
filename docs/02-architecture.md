# 02 – Architecture and Design Document

## 1. Architecture style

Domain-oriented microservices with service-owned PostgreSQL databases, synchronous REST for queries/commands that require immediate validation, and Kafka for asynchronous workflow events.

This is a logical architecture, not a claim that every interaction must be asynchronous. Keep the number of services manageable and introduce boundaries for a clear ownership reason.

## 2. Service catalogue

| Service | Owns | Responsibilities |
|---|---|---|
| API Gateway | Routing configuration | Entry point, JWT validation integration, rate limiting, routing |
| Discovery Server | Service registry | Local service discovery if Eureka is used |
| User Service | user_db | Registration, profile, verification, credentials/session metadata |
| Account Service | account_db | Account lifecycle, holds, available balance, account eligibility |
| Beneficiary Service | beneficiary_db | Beneficiary records and verification state |
| Transfer Service | transfer_db | Transfer request, idempotency, Saga orchestration, outbox |
| Ledger Service | ledger_db | Posting requests, immutable journals and balance validation |
| Transaction Service | transaction_db | Customer-facing transaction projection/history |
| Notification Service | notification_db | Notification requests, preferences and delivery attempts |
| Admin Service | admin_db | Admin operations, approvals, audit/investigation views |

### Ownership rules
- A service is the only writer and reader of its own database.
- Other services use APIs or events, not direct SQL against another service's tables.
- Do not share JPA entities across services.
- Common libraries may contain stable event contracts, error conventions and technical utilities, but not shared business state.

## 3. High-level request flow

1. Mobile/Web client calls API Gateway with access token.
2. Gateway routes request to Transfer Service.
3. Transfer Service validates request and idempotency key.
4. Transfer Service creates transfer and Saga state in one local database transaction.
5. Saga requests source-fund reservation from Account Service.
6. Account Service validates account state and atomically reserves available funds.
7. Transfer Service requests a ledger posting with a stable posting reference.
8. Ledger Service atomically creates a balanced journal and publishes `ledger.posted` through its outbox.
9. Transfer Service consumes confirmation, finalizes workflow and emits `transfer.completed`.
10. Transaction and Notification services consume completion event idempotently.
11. Reconciliation checks workflow, hold and ledger consistency.

## 4. Transfer Saga design

The Transfer Service is the orchestrator.

| Step | Command / action | Success | Definitive failure |
|---|---|---|---|
| 1 | Create transfer | Persist RECEIVED | Return validation error |
| 2 | Validate accounts/limits | Continue | REJECTED |
| 3 | Reserve source funds | FUNDS_RESERVED | FAILED |
| 4 | Request ledger posting | Await outcome | Investigate / compensate only when no posting exists |
| 5 | Confirm ledger posted | COMPLETED | Retry/query by posting reference |
| 6 | Finalize reservation | Hold consumed/released according to account model | Recovery workflow |
| 7 | Publish completion | Downstream projections update | Outbox retries |

### Critical uncertainty rule
If the Ledger Service times out, the posting may still have committed. Transfer Service must query Ledger Service using the unique posting reference or safely retry the same idempotent command. It must not release the source hold until it knows whether the posting exists and what state it is in.

## 5. Account reservation model

Account Service is authoritative for available balance and holds. A reservation is a durable record associated with transfer ID.

Illustrative invariant:
`available_balance = ledger_balance - active_holds - other_restrictions` (the exact model must be defined and consistently maintained).

A hold operation must be atomic. Use a conditional SQL update or row lock in PostgreSQL. Redis locks are not a substitute for database correctness.

## 6. Ledger model

Ledger Service stores immutable journal transactions and entries. A posting must balance by currency:
`SUM(debits) = SUM(credits)` for each journal and currency.

For a simple internal transfer, the conceptual journal contains a debit to the sender's customer liability account and a credit to the receiver's customer liability account. Accounting conventions should be confirmed for the chosen ledger model. The journal transaction and all entries are committed atomically.

## 7. Messaging patterns

- Transactional outbox for DB state + event publication.
- At-least-once delivery with idempotent consumers.
- Stable event ID and aggregate/transfer ID.
- Retry only transient failures.
- DLQ for exhausted or poison messages.
- Schema versioning and backward compatibility.
- Kafka key chosen to preserve ordering for a transfer within a topic.
- No claim of global exactly-once processing across Kafka and multiple databases.

## 8. Caching

Redis may cache customer/profile/reference data and support distributed rate limiting. Do not use cached balances as authoritative financial data. Define TTL and invalidation. If Redis is unavailable, safe read paths may fall back to the owning database.

## 9. Deployment topology

Local development:
- One PostgreSQL container with separate logical databases/schemas per service for convenience.
- One single-node Kafka KRaft broker.
- One Redis instance.
- Services run locally from IDE or Maven.

Production-like future topology:
- Separate managed/high-availability database instances per service where justified.
- Kafka cluster with replication and monitoring.
- Redis HA if required.
- Service-to-service TLS/mTLS, secrets manager, centralized telemetry and deployment orchestration.

## 10. Architecture risks

| Risk | Mitigation |
|---|---|
| Duplicate events | Consumer inbox/dedup table and idempotent handlers |
| DB committed, event not published | Transactional outbox |
| Uncertain ledger timeout | Query/retry by stable posting reference |
| Concurrent overspending | Atomic DB reservation and concurrency tests |
| Saga stuck after restart | Persist Saga state and scheduled recovery |
| Projection lag | Expose pending status and monitor consumer lag |
| Poison event | Bounded retry, DLQ and operational replay |
| Cross-service schema coupling | API/event contracts, no shared DB access |
