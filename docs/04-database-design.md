# 04 – Database Design

## 1. Conventions

- PostgreSQL.
- UUID primary keys.
- `NUMERIC(19,4)` for monetary values (confirm currency precision rules).
- `TIMESTAMPTZ` for timestamps, stored/handled consistently in UTC.
- Flyway migrations owned by each service.
- Foreign keys only within a service-owned database boundary.
- Unique constraints for business references.
- Indexes designed from query patterns and validated with `EXPLAIN ANALYZE`.
- Use soft deletion only where business history requires it; financial ledger entries are immutable.

## 2. Database ownership and tables

### user_db – User Service

`users`
- id UUID PK
- first_name VARCHAR(100) NOT NULL
- last_name VARCHAR(100) NOT NULL
- email VARCHAR(255) NOT NULL UNIQUE
- mobile_number VARCHAR(30) NOT NULL UNIQUE
- password_hash VARCHAR(255) NOT NULL
- status VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`user_verifications`
- id UUID PK
- user_id UUID NOT NULL
- verification_type VARCHAR(30) NOT NULL
- destination_masked VARCHAR(100)
- otp_hash VARCHAR(255) NOT NULL
- expires_at TIMESTAMPTZ NOT NULL
- verified_at TIMESTAMPTZ
- attempts INT NOT NULL DEFAULT 0
- created_at TIMESTAMPTZ NOT NULL

`refresh_tokens`
- id UUID PK
- user_id UUID NOT NULL
- token_hash VARCHAR(255) NOT NULL
- expires_at TIMESTAMPTZ NOT NULL
- revoked_at TIMESTAMPTZ
- replaced_by_token_id UUID
- created_at TIMESTAMPTZ NOT NULL

### account_db – Account Service

`accounts`
- id UUID PK
- user_id UUID NOT NULL (external reference; no cross-DB FK)
- account_number VARCHAR(40) NOT NULL UNIQUE
- account_type VARCHAR(30) NOT NULL
- currency CHAR(3) NOT NULL
- status VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`account_balances`
- account_id UUID PK
- ledger_balance NUMERIC(19,4) NOT NULL
- available_balance NUMERIC(19,4) NOT NULL
- version BIGINT NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`account_holds`
- id UUID PK
- account_id UUID NOT NULL
- transfer_id UUID NOT NULL UNIQUE
- amount NUMERIC(19,4) NOT NULL CHECK (amount > 0)
- currency CHAR(3) NOT NULL
- status VARCHAR(30) NOT NULL
- expires_at TIMESTAMPTZ
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

### beneficiary_db – Beneficiary Service

`beneficiaries`
- id UUID PK
- user_id UUID NOT NULL
- beneficiary_name VARCHAR(150) NOT NULL
- account_identifier_ciphertext TEXT NOT NULL
- bank_code VARCHAR(30)
- currency CHAR(3)
- status VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL
- deleted_at TIMESTAMPTZ

### transfer_db – Transfer Service

`transfers`
- id UUID PK
- transfer_reference VARCHAR(80) NOT NULL UNIQUE
- customer_id UUID NOT NULL
- source_account_id UUID NOT NULL
- destination_account_id UUID NOT NULL
- amount NUMERIC(19,4) NOT NULL CHECK (amount > 0)
- currency CHAR(3) NOT NULL
- status VARCHAR(30) NOT NULL
- description VARCHAR(255)
- idempotency_key VARCHAR(255) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`idempotency_records`
- id UUID PK
- customer_id UUID NOT NULL
- idempotency_key VARCHAR(255) NOT NULL
- request_hash VARCHAR(128) NOT NULL
- transfer_id UUID
- response_status INT
- response_body JSONB
- state VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- expires_at TIMESTAMPTZ
- UNIQUE(customer_id, idempotency_key)

`transfer_sagas`
- id UUID PK
- transfer_id UUID NOT NULL UNIQUE
- current_step VARCHAR(50) NOT NULL
- status VARCHAR(30) NOT NULL
- retry_count INT NOT NULL DEFAULT 0
- last_error TEXT
- started_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`transfer_saga_steps`
- id UUID PK
- saga_id UUID NOT NULL
- step_name VARCHAR(50) NOT NULL
- step_status VARCHAR(30) NOT NULL
- request_reference VARCHAR(100)
- started_at TIMESTAMPTZ
- completed_at TIMESTAMPTZ
- error_message TEXT

`outbox_events`
- id UUID PK
- aggregate_type VARCHAR(100) NOT NULL
- aggregate_id VARCHAR(100) NOT NULL
- event_type VARCHAR(150) NOT NULL
- schema_version INT NOT NULL
- payload JSONB NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- published_at TIMESTAMPTZ
- status VARCHAR(30) NOT NULL
- attempts INT NOT NULL DEFAULT 0

### ledger_db – Ledger Service

`ledger_transactions`
- id UUID PK
- posting_reference VARCHAR(100) NOT NULL UNIQUE
- transfer_id UUID NOT NULL
- status VARCHAR(30) NOT NULL
- currency CHAR(3) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- posted_at TIMESTAMPTZ

`ledger_entries`
- id UUID PK
- ledger_transaction_id UUID NOT NULL
- account_id UUID NOT NULL
- entry_type VARCHAR(10) NOT NULL CHECK (entry_type IN ('DEBIT','CREDIT'))
- amount NUMERIC(19,4) NOT NULL CHECK (amount > 0)
- currency CHAR(3) NOT NULL
- created_at TIMESTAMPTZ NOT NULL

`processed_commands`
- command_id UUID PRIMARY KEY
- command_type VARCHAR(100) NOT NULL
- request_hash VARCHAR(128) NOT NULL
- result JSONB
- processed_at TIMESTAMPTZ NOT NULL

`outbox_events`
- same outbox pattern as Transfer Service

Important: enforce journal balancing in Ledger Service transaction logic and consider a deferred constraint trigger if database-level enforcement is desired. A simple row-level CHECK cannot compare sums across rows.

### transaction_db – Transaction Service

`transactions`
- id UUID PK
- transfer_id UUID NOT NULL UNIQUE
- transaction_reference VARCHAR(100) NOT NULL UNIQUE
- customer_id UUID NOT NULL
- account_id UUID NOT NULL
- direction VARCHAR(10) NOT NULL
- amount NUMERIC(19,4) NOT NULL
- currency CHAR(3) NOT NULL
- status VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- updated_at TIMESTAMPTZ NOT NULL

`transaction_status_history`
- id UUID PK
- transaction_id UUID NOT NULL
- old_status VARCHAR(30)
- new_status VARCHAR(30) NOT NULL
- event_id UUID
- changed_at TIMESTAMPTZ NOT NULL

`processed_events`
- event_id UUID NOT NULL
- consumer_name VARCHAR(100) NOT NULL
- processed_at TIMESTAMPTZ NOT NULL
- PRIMARY KEY(event_id, consumer_name)

### notification_db – Notification Service

`notifications`
- id UUID PK
- event_id UUID NOT NULL
- recipient_id UUID NOT NULL
- channel VARCHAR(20) NOT NULL
- template_code VARCHAR(80) NOT NULL
- status VARCHAR(30) NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- sent_at TIMESTAMPTZ
- UNIQUE(event_id, recipient_id, channel, template_code)

`notification_delivery_attempts`
- id UUID PK
- notification_id UUID NOT NULL
- attempt_number INT NOT NULL
- provider_status VARCHAR(50)
- error_message TEXT
- attempted_at TIMESTAMPTZ NOT NULL

`notification_preferences`
- id UUID PK
- user_id UUID NOT NULL
- channel VARCHAR(20) NOT NULL
- enabled BOOLEAN NOT NULL
- updated_at TIMESTAMPTZ NOT NULL
- UNIQUE(user_id, channel)

### admin_db – Admin Service

`audit_logs`
- id UUID PK
- actor_id UUID NOT NULL
- actor_role VARCHAR(50) NOT NULL
- action VARCHAR(100) NOT NULL
- entity_type VARCHAR(100) NOT NULL
- entity_id VARCHAR(100) NOT NULL
- reason TEXT
- correlation_id VARCHAR(100)
- created_at TIMESTAMPTZ NOT NULL

`approval_requests`
- id UUID PK
- action_type VARCHAR(80) NOT NULL
- entity_id VARCHAR(100) NOT NULL
- requested_by UUID NOT NULL
- approved_by UUID
- status VARCHAR(30) NOT NULL
- reason TEXT NOT NULL
- created_at TIMESTAMPTZ NOT NULL
- decided_at TIMESTAMPTZ

## 3. Index strategy

Initial candidates:
- `transfers(transfer_reference)` unique.
- `transfers(customer_id, created_at DESC)`.
- `transfers(source_account_id, created_at DESC)`.
- `transfers(status, updated_at)`.
- `ledger_transactions(posting_reference)` unique.
- `ledger_entries(account_id, created_at DESC)`.
- `account_holds(account_id, status)`.
- `outbox_events(status, created_at)`.
- `transaction_status_history(transaction_id, changed_at)`.

Validate actual indexes against query plans and data volume.

## 4. Migration layout

Each service:
```text
src/main/resources/db/migration/
  V1__create_initial_tables.sql
  V2__add_transfer_status_history.sql
```

Never edit a migration that has already been applied in a shared environment. Add a new migration.
