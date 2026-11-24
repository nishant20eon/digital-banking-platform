# 05 – REST API Contracts

## 1. API conventions

- Base path: `/api/v1`.
- JSON request/response.
- UUID resource IDs.
- ISO-8601 timestamps.
- Money represented as decimal strings or JSON numbers with documented precision; Java uses `BigDecimal`.
- Pagination: `page`, `size`, and stable sorting.
- Authentication: `Authorization: Bearer <access-token>`.
- Transfer initiation requires `Idempotency-Key`.
- Use consistent error envelope.
- Do not expose internal stack traces or sensitive identifiers.

## 2. Standard error response

```json
{
  "timestamp": "2026-11-23T10:30:00Z",
  "status": 409,
  "errorCode": "IDEMPOTENCY_KEY_REUSED",
  "message": "The idempotency key was previously used with a different request.",
  "path": "/api/v1/transfers",
  "correlationId": "corr-example"
}
```

## 3. User and authentication APIs

| Method | Path | Purpose |
|---|---|---|
| POST | `/users/register` | Register |
| POST | `/users/verify` | Verify contact |
| GET | `/users/me` | Get current profile |
| PATCH | `/users/me` | Update profile |
| POST | `/auth/login` | Login |
| POST | `/auth/refresh` | Rotate refresh token |
| POST | `/auth/logout` | Revoke session |

Registration example:
```json
{
  "firstName": "Alex",
  "lastName": "Morgan",
  "email": "alex@example.test",
  "mobileNumber": "+910000000000",
  "password": "Use-a-local-test-secret-only"
}
```

## 4. Account APIs

| Method | Path | Purpose |
|---|---|---|
| POST | `/accounts` | Create account |
| GET | `/accounts` | List current customer's accounts |
| GET | `/accounts/{accountId}` | Account details |
| GET | `/accounts/{accountId}/balance` | Balance view |

Account creation example:
```json
{
  "accountType": "SAVINGS",
  "currency": "INR"
}
```

## 5. Beneficiary APIs

| Method | Path | Purpose |
|---|---|---|
| POST | `/beneficiaries` | Add beneficiary |
| GET | `/beneficiaries` | List beneficiaries |
| GET | `/beneficiaries/{id}` | Get beneficiary |
| PATCH | `/beneficiaries/{id}` | Update beneficiary |
| DELETE | `/beneficiaries/{id}` | Deactivate beneficiary |

## 6. Transfer APIs

| Method | Path | Purpose |
|---|---|---|
| POST | `/transfers` | Initiate transfer |
| GET | `/transfers/{transferId}` | Get transfer status |
| POST | `/transfers/{transferId}/cancel` | Request cancellation if allowed |

Request:
```http
POST /api/v1/transfers
Authorization: Bearer <token>
Idempotency-Key: 3f4c0b1c-example
Content-Type: application/json
```

```json
{
  "sourceAccountId": "11111111-1111-1111-1111-111111111111",
  "destinationAccountId": "22222222-2222-2222-2222-222222222222",
  "amount": "5000.00",
  "currency": "INR",
  "description": "Personal transfer"
}
```

Accepted response:
```json
{
  "transferId": "33333333-3333-3333-3333-333333333333",
  "transferReference": "TRF-EXAMPLE",
  "status": "RECEIVED",
  "amount": "5000.00",
  "currency": "INR",
  "createdAt": "2026-11-23T10:00:00Z"
}
```

Use HTTP 202 when the operation is accepted for asynchronous processing. Do not label it successful/completed until ledger confirmation.

## 7. Transaction APIs

| Method | Path | Purpose |
|---|---|---|
| GET | `/transactions` | List current customer's transaction projection |
| GET | `/transactions/{transactionId}` | Transaction details |

Supported filters: `accountId`, `from`, `to`, `status`, `direction`, `page`, `size`, `sort`.

## 8. Admin APIs

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/transfers` | Search transfers |
| GET | `/admin/transfers/{id}/workflow` | Inspect Saga steps |
| POST | `/admin/accounts/{id}/freeze` | Freeze account |
| POST | `/admin/accounts/{id}/unfreeze` | Unfreeze account |
| POST | `/admin/transfers/{id}/reversals` | Request reversal |
| GET | `/admin/reconciliation/exceptions` | Review exceptions |

High-risk operations require explicit authorization, reason capture, audit and (where configured) maker-checker approval.

## 9. API security requirements

- Enforce ownership in the service, not only the gateway.
- Validate all path/query/body inputs.
- Apply rate limits to authentication and sensitive operations.
- Never accept customer ID from request body as proof of identity; derive caller identity from authenticated principal.
- Mask account identifiers in customer-facing responses.
- Use OpenAPI security definitions.
