# 00 – Project Charter

## 1. Purpose

Build a realistic, enterprise-style digital banking backend as a portfolio and engineering learning project. The platform demonstrates service boundaries, financial data integrity, asynchronous messaging, distributed workflow recovery, security and automated testing.

## 2. Vision

A customer can register, create accounts, add beneficiaries, transfer funds between eligible platform accounts, and inspect transaction status/history. Operations users can investigate failures and initiate controlled reversals subject to authorization.

## 3. Goals

1. Implement independently deployable Spring Boot services.
2. Enforce service-owned persistence.
3. Implement a balanced, immutable double-entry ledger.
4. Protect transfer initiation against duplicate requests and concurrent spending.
5. Use Kafka for asynchronous events.
6. Implement transactional outbox, consumer deduplication, retries and DLQ.
7. Implement durable Saga orchestration and compensation.
8. Provide tests, API documentation, observability and local reproducibility.

## 4. Non-goals for initial releases

- Real deposits, withdrawals or external bank transfers.
- Production payment gateway or card processing.
- Foreign exchange.
- Lending, interest calculation or overdraft.
- Real KYC/AML provider integrations.
- Regulatory certification or production banking operation.
- Native mobile or web frontends.

## 5. Actors

| Actor | Description |
|---|---|
| Customer | Registers, manages own accounts and initiates transfers |
| Support agent | Reads permitted customer/transfer information |
| Operations user | Investigates pending/failed workflows |
| Administrator | Manages configuration and account restrictions |
| System service | Executes workflow steps and publishes/consumes events |

## 6. Success criteria

- A happy-path transfer produces balanced ledger entries exactly once from the business perspective.
- Repeated API requests with the same idempotency key do not create another transfer.
- Concurrent requests cannot spend more than available funds.
- A service restart does not lose durable Saga state.
- Duplicate Kafka delivery does not duplicate business effects.
- Failed or uncertain posting outcomes are recoverable and reconcilable.
- All critical business paths have automated tests.
- A new developer can run the infrastructure and follow the sprint backlog using this repository.

## 7. Key principles

- Correctness before throughput.
- Ledger is the financial source of truth.
- Explicit state machines over ad hoc status updates.
- Durable workflows over in-memory orchestration.
- At-least-once messaging plus idempotent processing.
- Least-privilege security.
- Observable failure and safe recovery.
