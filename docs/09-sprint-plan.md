# 09 – Sprint Plan and Release Roadmap

## Planning assumptions

- 2-week sprints.
- One developer or a small learning team.
- Story points are intentionally omitted until the team estimates complexity.
- Sprint goals are proposed sequencing, not fixed delivery commitments.
- Every sprint includes tests, documentation and a working increment.

## Sprint 0 – Repository and engineering foundation

**Goal:** A reproducible developer environment.

Stories:
- S0-01 Create Git repository and directory layout.
- S0-02 Configure root Maven dependency management, Java 25, Spring Boot 4.2.0 and Spring Cloud 2026.0.0.
- S0-03 Add Docker Compose for PostgreSQL 18.7, Kafka 4.4.0 KRaft and Redis 8.2.
- S0-04 Add `.env.example`, `.gitignore` and local setup guide.
- S0-05 Define code formatting, package naming and branch strategy.
- S0-06 Add CI build/test workflow.

Exit criteria:
- A new clone can start local infrastructure.
- Maven build runs.
- No secrets are committed.

## Sprint 1 – Gateway, discovery and service skeletons

**Goal:** Establish deployable service foundations.

Stories:
- S1-01 Create API Gateway module.
- S1-02 Create Eureka discovery module (if retaining Eureka).
- S1-03 Create health/readiness endpoints for services.
- S1-04 Add common error response and correlation ID conventions.
- S1-05 Add OpenAPI baseline.
- S1-06 Add service configuration profiles.

Exit criteria:
- Gateway routes to a sample service.
- Health checks work.
- Local configuration is documented.

## Sprint 2 – User registration and authentication

**Goal:** Secure customer identity lifecycle.

Stories:
- S2-01 Register customer.
- S2-02 Verify email/mobile using a development stub.
- S2-03 Login and issue JWT.
- S2-04 Refresh-token rotation and logout/revocation.
- S2-05 Profile read/update.
- S2-06 Role and authorization tests.

Exit criteria:
- Registration and authentication APIs work.
- Passwords and refresh tokens are not stored in plaintext.
- Authorization tests pass.

## Sprint 3 – Account management

**Goal:** Establish customer account ownership and balance model.

Stories:
- S3-01 Create account.
- S3-02 List and retrieve own accounts.
- S3-03 Implement account states.
- S3-04 Implement balance model.
- S3-05 Implement account freeze/unfreeze authorization.
- S3-06 Add account migration and integration tests.

Exit criteria:
- Ownership is enforced.
- No floating-point money types.
- Balance mutation paths are controlled.

## Sprint 4 – Beneficiaries and transfer foundation

**Goal:** Prepare transfer initiation.

Stories:
- S4-01 Add beneficiary.
- S4-02 List/update/deactivate beneficiary.
- S4-03 Create transfer API and request validation.
- S4-04 Implement idempotency records.
- S4-05 Add transfer and Saga persistence.
- S4-06 Implement transfer query/status API.

Exit criteria:
- Duplicate request behavior is tested.
- Transfer is durably accepted but not yet financially completed.

## Sprint 5 – Holds and concurrency control

**Goal:** Prevent overspending.

Stories:
- S5-01 Reserve source funds.
- S5-02 Release reserved funds.
- S5-03 Consume/finalize reservation after posting.
- S5-04 Implement atomic database concurrency control.
- S5-05 Add duplicate command protection.
- S5-06 Add concurrent transfer integration tests.

Exit criteria:
- Concurrent transfers cannot reserve more than available funds.
- Reserve/release operations are idempotent.

## Sprint 6 – Ledger and transaction history

**Goal:** Implement financial source of truth.

Stories:
- S6-01 Create ledger journal and entry schema.
- S6-02 Implement balanced posting.
- S6-03 Enforce unique posting reference.
- S6-04 Make posted entries immutable.
- S6-05 Implement transaction projection/history.
- S6-06 Implement reversal journal design and tests.

Exit criteria:
- Posting is atomic and balanced.
- Duplicate posting does not duplicate entries.

## Sprint 7 – Kafka and transactional outbox

**Goal:** Reliable asynchronous communication.

Stories:
- S7-01 Define versioned event envelope.
- S7-02 Implement Transfer Service outbox.
- S7-03 Implement Ledger Service outbox.
- S7-04 Implement Kafka producer/consumer configuration.
- S7-05 Implement inbox/deduplication.
- S7-06 Add retry and DLQ handling.

Exit criteria:
- DB update and outbox insert are atomic.
- Duplicate event processing is safe.
- Failed events are observable.

## Sprint 8 – Saga orchestration and recovery

**Goal:** Complete distributed transfer lifecycle.

Stories:
- S8-01 Implement reserve → post → complete Saga.
- S8-02 Implement durable Saga step transitions.
- S8-03 Implement compensation/release workflow.
- S8-04 Implement timeout and uncertain-result query.
- S8-05 Implement stuck Saga recovery job.
- S8-06 Add fault-injection integration tests.

Exit criteria:
- Happy path completes.
- Definite failure compensates.
- Ambiguous ledger result is resolved by reference lookup.
- Restart does not lose workflow state.

## Sprint 9 – Notifications and admin operations

**Goal:** Operational usability.

Stories:
- S9-01 Consume transfer completion/failure events.
- S9-02 Create notification preference and delivery records.
- S9-03 Implement notification deduplication and retry.
- S9-04 Admin transaction search.
- S9-05 Admin Saga inspection.
- S9-06 Audit and approval workflow foundation.

Exit criteria:
- Notifications are not duplicated.
- Sensitive admin actions are authorized and audited.

## Sprint 10 – Reconciliation, security hardening and observability

**Goal:** Improve recoverability and operational insight.

Stories:
- S10-01 Implement reconciliation queries/jobs.
- S10-02 Add OpenTelemetry tracing.
- S10-03 Add metrics and dashboards.
- S10-04 Add security hardening and secret handling.
- S10-05 Add circuit breakers/timeouts where suitable.
- S10-06 Add operational runbooks.

Exit criteria:
- Stuck/mismatched records can be identified.
- Cross-service trace correlation works.
- Operational failure modes are documented.

## Sprint 11 – End-to-end, performance and deployment

**Goal:** Demonstrate a complete working system.

Stories:
- S11-01 Build end-to-end happy-path suite.
- S11-02 Build failure/recovery suite.
- S11-03 Run concurrency and load tests.
- S11-04 Add CI pipeline and container builds.
- S11-05 Document deployment and recovery.
- S11-06 Final architecture and demo documentation.

Exit criteria:
- A clean environment can be deployed from documented steps.
- Test evidence and known limitations are documented.

## Release gates

### Release 1 – Core banking learning MVP
User/auth, account, beneficiary, transfer API, holds, ledger and basic history.

### Release 2 – Distributed systems
Kafka, outbox, idempotency, retries, DLQ, Saga, compensation and recovery.

### Release 3 – Operational maturity
Admin, audit, reconciliation, observability, performance and deployment.

## Sprint review template

At the end of each sprint record:
- Completed stories.
- Demo scenario.
- Tests executed and results.
- Database/API/event changes.
- Known defects and risks.
- Decisions/ADRs added.
- Stories carried forward.
