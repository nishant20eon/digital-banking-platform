# ADR-0003: Saga orchestration and transactional outbox

- Status: Accepted
- Date: 2026-11-23

## Context
Transfer workflow spans Account, Ledger, Transfer, Transaction and Notification services. A single ACID transaction cannot cover all independent databases and Kafka.

## Decision
Transfer Service orchestrates a durable Saga. Each service uses a transactional outbox for events emitted after local state changes. Consumers use inbox/deduplication records and idempotent handlers. Compensation is an explicit durable workflow.

## Consequences
- Eventual consistency is visible to clients as PROCESSING/pending states.
- Every command needs a stable reference.
- Unknown outcomes require query/retry by reference.
- Reconciliation is required.
- No claim of global exactly-once processing.

## Revisit when
Workflow complexity or throughput requires a different orchestration engine or choreography model.
