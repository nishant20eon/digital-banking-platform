# 08 – Testing Strategy

## 1. Test pyramid

1. Unit tests – domain logic, validators, state transitions.
2. Slice tests – MVC, JPA, security configuration.
3. Integration tests – PostgreSQL, Kafka, Redis via Testcontainers.
4. Contract tests – REST and event schema compatibility.
5. End-to-end tests – complete transfer workflows.
6. Performance/resilience tests – concurrency, throughput, failure recovery.

## 2. Unit test requirements

### User Service
- Duplicate email/mobile.
- Password hashing.
- Verification expiry and attempt limits.
- Login lockout/session revocation.

### Account Service
- Account ownership and status.
- Reservation success/insufficient balance.
- Duplicate reserve/release command.
- Concurrent reservation.
- Hold state transitions.

### Transfer Service
- Amount/currency validation.
- Idempotency same key/same payload.
- Idempotency same key/different payload.
- Saga permitted and forbidden transitions.
- Timeout and uncertain-result handling.
- Compensation behavior.

### Ledger Service
- Balanced journal accepted.
- Unbalanced journal rejected.
- Duplicate posting reference returns original result.
- Same posting reference/different payload rejected.
- Posted entries immutable.
- Duplicate reversal prevented.

### Notification Service
- Duplicate event does not send duplicate notification.
- Retryable vs permanent provider error.
- Preference handling.

## 3. Integration tests

Use Testcontainers for PostgreSQL, Kafka and Redis where needed.

Verify:
- Flyway migrations apply cleanly.
- JPA mappings and constraints.
- Transaction rollback.
- Conditional balance update under concurrency.
- Outbox publisher recovery.
- Consumer deduplication and offset behavior.
- Retry/DLQ routing.
- Service restart with durable Saga state.

## 4. Critical end-to-end scenarios

| Scenario | Expected outcome |
|---|---|
| Valid transfer | One completed transfer and balanced journal |
| Insufficient balance | Rejected; no ledger posting |
| Duplicate API request | Same transfer result |
| Same idempotency key, changed payload | HTTP 409 |
| Two concurrent transfers exceed balance | At most affordable reservation succeeds |
| Destination account invalid | Rejected before posting |
| Account frozen before reserve | Reservation rejected |
| Kafka temporarily unavailable | Outbox retains event and retries |
| Duplicate Kafka delivery | No duplicate business effect |
| Ledger request times out after commit | Query/retry by posting reference; no duplicate posting |
| Ledger definitive rejection | Compensation releases hold |
| Release command delivered twice | One release effect |
| Transfer Service restarts | Saga resumes from persisted state |
| DLQ replay | Audited and deduplicated |
| Reversal requested twice | One valid reversal |

## 5. Performance and concurrency

Use Gatling or JMeter. Test:
- Concurrent transfers against one source account.
- High-volume account history reads.
- Kafka consumer lag under load.
- Database connection pool saturation.
- Outbox backlog recovery.
- API latency under dependency degradation.

Do not claim capacity from a test run without documenting machine size, dataset, configuration, duration and percentile metrics.

## 6. Test data

Use synthetic customers, accounts and transactions. Never use real personal or banking data.
