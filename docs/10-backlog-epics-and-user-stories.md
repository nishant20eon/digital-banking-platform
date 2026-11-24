# 10 – Product Backlog: Epics and User Stories

Each story must be refined into implementation tasks and test cases before development. Acceptance criteria are the minimum expected behavior.

## EPIC E01 – Customer identity and authentication

### US-101 Register customer
**As a** new customer, **I want** to register, **so that** I can use the platform.

Acceptance criteria:
- Email and mobile are unique.
- Required fields and formats are validated.
- Password is hashed using an approved password encoder.
- Customer starts in a verification-pending state.
- Registration is auditable.
- Duplicate registration returns a safe conflict response.

DB: `users`, `user_verifications`.
Code: controller, request/response DTOs, domain/service, repository, password encoder, validation, migration.
API: `POST /api/v1/users/register`.
Tests: valid registration, duplicate email/mobile, invalid fields, password not persisted in plaintext.

### US-102 Verify contact
- Generate a one-time code through a development notification adapter.
- Store only a hash.
- Enforce expiry and attempt limits.
- Successful verification changes user state.
DB: verification attempts and verified timestamp.
API: `POST /api/v1/users/verify`.

### US-103 Login
- Authenticate credentials.
- Reject disabled/unverified users according to policy.
- Issue short-lived access token and refresh token.
- Apply brute-force protection.
API: `POST /api/v1/auth/login`.

### US-104 Refresh/revoke session
- Rotate refresh token.
- Store token hash.
- Revoke old token and support logout.
API: `POST /api/v1/auth/refresh`, `POST /api/v1/auth/logout`.

### US-105 Manage profile
- Read and update own profile.
- Require re-verification for sensitive contact changes.
API: `GET/PATCH /api/v1/users/me`.

## EPIC E02 – Account management

### US-201 Create account
- Only eligible customer can create account.
- Unique account number.
- Currency and account type required.
- Initial balance is zero.
DB: `accounts`, `account_balances`.
API: `POST /api/v1/accounts`.

### US-202 List/view accounts
- Customer sees only own accounts.
- Mask account number.
- Show status and balances.
API: `GET /api/v1/accounts`, `GET /api/v1/accounts/{id}`.

### US-203 Freeze/unfreeze account
- Authorized operations/admin role only.
- Reason required.
- Audit state transition.
- Frozen account cannot initiate new reservations.
API: admin freeze/unfreeze endpoints.

### US-204 Close account
- No active holds or unresolved transfers.
- Balance must meet closure policy.
- Retain required historical records.
- Closure is auditable.

## EPIC E03 – Beneficiary management

### US-301 Add beneficiary
- Validate required details.
- Protect sensitive account identifier.
- Associate with authenticated customer.
- Apply verification policy.
DB: `beneficiaries`.
API: `POST /api/v1/beneficiaries`.

### US-302 List/view beneficiaries
- Return only customer's own records.
- Support pagination.
API: `GET /api/v1/beneficiaries`.

### US-303 Update/deactivate beneficiary
- Validate ownership.
- Audit changes.
- Do not break historical transfer references.
API: `PATCH/DELETE /api/v1/beneficiaries/{id}`.

## EPIC E04 – Transfer initiation and lifecycle

### US-401 Initiate internal transfer
- Authenticated customer.
- Source account belongs to customer.
- Source and destination eligible.
- Same currency in initial release.
- Positive amount and configured limits.
- Idempotency key required.
- Durable transfer and Saga created.
DB: `transfers`, `idempotency_records`, `transfer_sagas`.
API: `POST /api/v1/transfers`.

### US-402 Validate transfer
- Check ownership, status, currency, amount, limits and destination eligibility.
- Reject same-account transfer by default.
- Return stable business error codes.

### US-403 Query transfer
- Customer can query own transfer.
- Status and reference returned.
API: `GET /api/v1/transfers/{id}`.

### US-404 Cancel eligible transfer
- Cancellation only before defined irreversible step.
- Release hold through durable workflow.
- Reject cancellation after posting.
API: `POST /api/v1/transfers/{id}/cancel`.

### US-405 Scheduled transfer (later release)
- Store schedule and recurrence.
- Revalidate at execution.
- Prevent duplicate execution.

## EPIC E05 – Funds reservation and concurrency

### US-501 Reserve funds
- Atomic conditional update or row lock.
- Create durable hold with transfer reference.
- Duplicate reserve request returns original result.
- Insufficient balance rejects reservation.
DB: `account_holds`, `account_balances`.

### US-502 Release funds
- Only release an active hold.
- Duplicate release is safe.
- Record reason and event.
- Do not release if ledger outcome is uncertain.

### US-503 Finalize/consume hold
- Finalize after confirmed ledger posting.
- Ensure available/ledger balance semantics are consistent.
- Duplicate finalization is safe.

### US-504 Prevent concurrent overspending
- Use PostgreSQL concurrency control.
- Test two or more simultaneous transfers.
- Verify total reserved amount never exceeds spendable funds.

## EPIC E06 – Ledger and transaction history

### US-601 Post balanced journal
- Unique posting reference.
- Debit and credit lines balance per currency.
- Journal and entries commit atomically.
- Duplicate same command returns existing outcome.
- Same reference/different payload is conflict.
DB: `ledger_transactions`, `ledger_entries`, `processed_commands`.

### US-602 Enforce immutability
- No update/delete of posted journal entries.
- Corrections use compensating postings.
- Audit all administrative actions.

### US-603 Transaction history
- Build customer-facing projection from events.
- Filter, sort and paginate.
- Deduplicate incoming events.
DB: `transactions`, `transaction_status_history`, `processed_events`.
API: `GET /api/v1/transactions`.

### US-604 Reverse transaction
- Validate original posting and reversal eligibility.
- Create compensating journal.
- Link reversal to original.
- Prevent duplicate reversal.
- Require authorization/reason/approval.

### US-605 Reconcile transfers and ledger
- Detect missing postings, duplicate references, stale holds and mismatched status.
- Create durable exception records.
- Never silently modify financial records.

## EPIC E07 – Kafka and reliable messaging

### US-701 Define event contracts
- Common envelope.
- Schema version.
- Event ID, aggregate ID, correlation ID and timestamp.
- Compatibility policy.

### US-702 Transactional outbox
- Persist business update and outbox event in same local transaction.
- Publisher retries unpublished events.
- Monitor backlog.
DB: `outbox_events`.

### US-703 Idempotent consumers
- Persist inbox marker and business effect atomically.
- Duplicate event creates no duplicate effect.
DB: `processed_events` or equivalent.

### US-704 Retry and DLQ
- Bounded retry with backoff.
- Distinguish transient/permanent failures.
- Preserve metadata in DLQ.
- Authorized, audited replay.

## EPIC E08 – Saga orchestration and recovery

### US-801 Execute transfer Saga
- Persist each step.
- Reserve funds, post ledger, finalize and publish outcome.
- Transfer completion requires confirmed ledger posting.

### US-802 Compensate failed transfer
- Release hold only after confirming no successful ledger posting.
- Compensation has its own state and retries.
- Failure of compensation moves to manual review.

### US-803 Handle timeouts/unknown outcomes
- Query downstream by stable business reference.
- Retry idempotently.
- Do not interpret timeout as definitive failure.

### US-804 Recover stuck Saga
- Scheduled scanner finds stale nonterminal workflows.
- Reconcile step outcome.
- Resume safely.
- Emit operational metrics/alerts.

## EPIC E09 – Redis and rate limiting

### US-901 Cache safe read data
- TTL and invalidation defined.
- Database remains source of truth.
- No authoritative balance decision from cache.

### US-902 Distributed rate limiting
- Rate-limit login and sensitive APIs.
- Return 429 with retry guidance.
- Define behavior when Redis is unavailable.

## EPIC E10 – Notifications

### US-1001 Transfer initiated notification
### US-1002 Transfer completed notification
### US-1003 Transfer failed notification
### US-1004 Notification retry and DLQ
### US-1005 Notification preferences

Acceptance criteria across stories:
- Trigger from business events.
- Deduplicate by event/recipient/channel/template.
- Retry transient provider failures.
- Never include sensitive credentials or full account numbers.
- Mandatory/security notices may override user preferences according to policy.

## EPIC E11 – Admin, audit and approvals

### US-1101 Role-based admin access
### US-1102 Search transfers and transactions
### US-1103 Inspect Saga workflow and failure history
### US-1104 Freeze/unfreeze account with reason and audit
### US-1105 Request reversal with maker-checker approval
### US-1106 Search audit trail

Acceptance criteria:
- Least privilege.
- Sensitive actions are audited.
- High-risk operations require approval policy.
- Admin cannot silently edit ledger history.

## EPIC E12 – Observability, security and production readiness

### US-1201 Distributed tracing
- Trace context across REST and Kafka.

### US-1202 Centralized structured logging
- Correlation IDs and transfer references.
- No secrets or sensitive values.

### US-1203 Metrics and dashboards
- API latency/error rate, Kafka lag, outbox backlog, DLQ depth, stuck Saga, ledger failures and reconciliation exceptions.

### US-1204 Resilience
- Timeouts, bounded retries, circuit breakers and bulkheads where appropriate.

### US-1205 Disaster recovery
- Backup, restore, RPO/RTO, recovery and reconciliation runbooks.

### US-1206 Security hardening
- JWT validation, TLS, secrets, input validation, rate limiting and dependency scanning.

## EPIC E13 – Quality engineering

### US-1301 Unit tests
### US-1302 PostgreSQL/Testcontainers integration tests
### US-1303 Kafka integration tests
### US-1304 API/security tests
### US-1305 End-to-end workflow tests
### US-1306 Concurrency, performance and resilience tests

Every epic should be refined with estimates, dependencies and sprint assignment before execution.
