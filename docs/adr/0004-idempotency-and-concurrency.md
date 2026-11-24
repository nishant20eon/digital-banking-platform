# ADR-0004: Idempotency and concurrency control

- Status: Accepted
- Date: 2026-11-23

## Context
Clients retry requests, Kafka may redeliver messages and multiple transfers can race against the same account balance.

## Decision
Use customer-scoped API idempotency keys with request hashes and unique constraints. Use unique posting references for ledger commands. Use database-level atomic conditional updates/locking for funds reservation. Redis is not the source of truth for financial concurrency.

## Consequences
- Same key/same payload returns original operation.
- Same key/different payload is a conflict.
- Consumer effects and inbox marker must commit atomically.
- Concurrency integration tests are mandatory.

## Revisit when
The platform introduces multiple ledgers, external settlement rails or a different consistency model.
