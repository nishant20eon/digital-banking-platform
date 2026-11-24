# 01 – Product Requirements Document (PRD)

## 1. Product overview

The Digital Banking Platform is a backend system for customer and internal account management, internal transfers and transaction tracking.

## 2. User capabilities

### Customer
- Register and verify contact details.
- Authenticate and refresh/revoke sessions.
- View and update profile.
- Create and view eligible accounts.
- View available and ledger balances.
- Add, list, update and deactivate beneficiaries.
- Initiate internal transfers.
- Query transfer and transaction status.
- View paginated transaction history.
- Request cancellation only while a transfer is in a cancellable state.

### Operations / Admin
- Search transfers and transactions.
- Inspect Saga step history and correlation IDs.
- Freeze/unfreeze accounts subject to role and approval policy.
- Initiate reversal workflow subject to authorization and maker-checker policy.
- Inspect audit trail and reconciliation exceptions.
- Replay eligible DLQ messages through controlled tooling, never by blind bulk replay.

## 3. Functional requirements

### FR-01 Registration
The system shall register a customer with unique email and mobile number, store a password hash, and mark contact verification status.

### FR-02 Authentication
The system shall authenticate users, issue short-lived access tokens, rotate refresh tokens, revoke sessions and enforce role-based authorization.

### FR-03 Account lifecycle
The system shall create accounts for verified customers, assign unique account numbers, maintain account status and expose only accounts owned by the authenticated customer.

### FR-04 Balance model
The system shall distinguish ledger balance from available balance. Available balance shall account for holds and restrictions. Balance changes shall be protected by database transactions and concurrency control.

### FR-05 Beneficiaries
The system shall allow a customer to maintain recipient records. Sensitive account identifiers shall be encrypted or tokenized where appropriate. Beneficiary changes shall be audited.

### FR-06 Transfer initiation
The system shall validate source ownership, account eligibility, destination eligibility, amount, currency, limits, available funds and idempotency before creating a durable transfer workflow.

### FR-07 Fund reservation
The system shall reserve funds atomically and prevent concurrent spending of the same funds. A reservation shall have a durable reference to the transfer.

### FR-08 Ledger posting
The system shall post balanced debit and credit entries atomically in the Ledger Service database. A posting reference shall be unique. Posted entries shall be immutable.

### FR-09 Transaction history
The system shall expose paginated transaction history with filters for date, status and transaction type, subject to authorization.

### FR-10 Saga workflow
The system shall persist each workflow step, handle retries and timeouts, and execute compensating actions when a step definitively fails. It shall not compensate based only on an ambiguous timeout.

### FR-11 Messaging
The system shall publish domain events using an outbox, consume events idempotently, retry transient errors and route exhausted messages to a DLQ.

### FR-12 Notifications
The system shall create notification requests for relevant transfer events and avoid duplicate notifications for the same event/channel/template combination.

### FR-13 Administration
The system shall enforce role-based permissions, audit sensitive actions and support investigation of stuck or failed transfers.

### FR-14 Reconciliation
The system shall identify transfers whose workflow status, ledger postings and account reservations are inconsistent and create an investigation record.

## 4. Business rules

1. Initial release supports internal transfers only.
2. Source and destination accounts must use the same currency.
3. Transfer amount must be positive and conform to configured currency precision.
4. Source and destination accounts must be distinct unless an explicit business rule permits otherwise; default is reject same-account transfer.
5. Source account must belong to the authenticated customer.
6. Frozen, closed or otherwise restricted accounts cannot initiate transfers.
7. Destination account must be eligible to receive funds.
8. A transfer must have a stable unique transfer ID and posting reference.
9. A successful transfer must have equal total debits and credits per currency.
10. A committed ledger posting cannot be edited or deleted.
11. A reversal is a new compensating posting linked to the original posting.
12. A transfer must not be marked COMPLETED until ledger posting is confirmed.
13. A timeout does not prove that a remote operation failed.
14. Duplicate API requests and Kafka events must not duplicate financial effects.
15. Customer-facing responses must not disclose another customer's sensitive information.

## 5. Transfer statuses

See [Domain Model and State Machines](03-domain-model-and-state-machines.md). Initial status vocabulary:
`RECEIVED`, `VALIDATING`, `REJECTED`, `FUNDS_RESERVING`, `FUNDS_RESERVED`, `LEDGER_POSTING`, `COMPLETED`, `COMPENSATING`, `FAILED`, `CANCELLED`, `MANUAL_REVIEW`.

## 6. Acceptance baseline

A story is accepted only when:
- Acceptance criteria pass.
- Negative and boundary cases are tested.
- Authorization and ownership checks are tested.
- Database constraints and migration are included.
- Logs/metrics are added for operationally important flows.
- API/event contract documentation is updated.
- No secrets or sensitive data are logged.
- Recovery behavior is defined for asynchronous workflows.

## 7. Assumptions requiring product decision

- Currency support and precision rules.
- Transfer limits and fee model.
- Verification requirements.
- Whether transfers between a customer's own accounts are allowed.
- Cancellation cutoff and scheduled transfer behavior.
- Notification channels.
- Retention periods.
- Admin maker-checker thresholds.
- Whether account balance is maintained as a projection or synchronously updated by Account Service.
