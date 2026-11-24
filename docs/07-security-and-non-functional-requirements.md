# 07 – Security and Non-Functional Requirements

## 1. Authentication and authorization

- Spring Security resource-server JWT validation.
- Short-lived access tokens.
- Refresh token rotation, hashing at rest and revocation.
- Role-based authorization: CUSTOMER, SUPPORT_AGENT, OPERATIONS, ADMIN, SUPER_ADMIN.
- Method-level authorization for sensitive actions.
- Ownership checks in each owning service.
- Least-privilege service credentials.

## 2. Sensitive data

- BCrypt/Argon2 password hashing through approved password encoder.
- Encrypt sensitive beneficiary/account identifiers at rest using managed keys in real deployments.
- Never log secrets, passwords, OTPs, tokens, full account identifiers or payment credentials.
- Mask sensitive values in APIs and logs.
- Keep local synthetic test data only.

## 3. Service communication

- TLS for external traffic.
- Service-to-service authentication; mTLS for production-like environments where feasible.
- Store secrets outside source control.
- Rotate credentials and signing keys.
- Validate JWT issuer, audience, expiry and signature.

## 4. API protection

- Input validation and output encoding.
- Rate limiting and brute-force protection.
- Request size limits.
- CORS allow-list.
- Secure headers.
- Idempotency for transfer initiation.
- Protection against insecure direct object references (IDOR).
- Avoid detailed error leakage.

## 5. Financial integrity

- `BigDecimal` and PostgreSQL `NUMERIC`.
- Explicit currency scale/rounding policy.
- Atomic ledger posting.
- Immutable posted entries.
- Compensating reversals.
- Atomic funds reservation.
- Idempotent command processing.
- Reconciliation and operational exception handling.
- Never use Redis as authoritative balance storage.

## 6. Resilience

- Timeouts on remote calls.
- Bounded retries with backoff and jitter.
- Circuit breakers for suitable remote dependencies.
- Bulkheads where useful.
- Durable outbox and Saga state.
- Health/readiness/liveness endpoints.
- Graceful shutdown and Kafka consumer handling.

## 7. Observability

Logs should include:
- service name
- environment
- trace ID
- correlation ID
- transfer ID where applicable
- event ID
- outcome and error code
- duration

Metrics:
- request rate, error rate, latency
- active transfer counts by state
- Saga age and stuck-workflow count
- outbox backlog and publish failures
- Kafka consumer lag
- DLQ depth
- ledger posting failures
- reconciliation exceptions
- DB pool utilization

Use OpenTelemetry/Micrometer. Avoid high-cardinality metric labels such as raw customer IDs.

## 8. Suggested initial SLOs (to validate)

- Availability objective: 99.9% for a future production-like deployment.
- Read API latency: define p95/p99 per endpoint after baseline load testing.
- Transfer initiation response: acknowledge durable acceptance, not final settlement.
- Recovery objectives (RPO/RTO): explicitly decide before production deployment.
- No known unbalanced committed ledger journals.

These are planning targets, not achieved guarantees.

## 9. Compliance note

A real banking or payment product needs expert review of applicable banking, privacy, cybersecurity, KYC/AML, payment-network, consumer-protection, audit and data-retention obligations. This repository does not establish compliance.
