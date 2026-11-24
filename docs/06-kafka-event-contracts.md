# 06 – Kafka Event Contracts

## 1. Naming and envelope

Topic names use lowercase dot-separated domain names. Event envelope:

```json
{
  "eventId": "uuid",
  "eventType": "TRANSFER_INITIATED",
  "schemaVersion": 1,
  "occurredAt": "2026-11-23T10:00:00Z",
  "producer": "transfer-service",
  "aggregateType": "TRANSFER",
  "aggregateId": "transfer-uuid",
  "correlationId": "correlation-uuid",
  "causationId": "optional-event-or-command-id",
  "payload": {}
}
```

Use schema validation and compatibility rules. Never put passwords, OTPs, access tokens, full account numbers or unnecessary personal data in events.

## 2. Initial topic catalogue

| Topic | Producer | Consumer(s) |
|---|---|---|
| `user.registered.v1` | User | Notification |
| `user.verified.v1` | User | Account |
| `account.created.v1` | Account | Notification |
| `funds.reservation.requested.v1` | Transfer | Account |
| `funds.reserved.v1` | Account | Transfer |
| `funds.reservation.failed.v1` | Account | Transfer |
| `ledger.posting.requested.v1` | Transfer | Ledger |
| `ledger.posted.v1` | Ledger | Transfer |
| `ledger.posting.failed.v1` | Ledger | Transfer |
| `funds.release.requested.v1` | Transfer | Account |
| `funds.released.v1` | Account | Transfer |
| `transfer.completed.v1` | Transfer | Transaction, Notification |
| `transfer.failed.v1` | Transfer | Transaction, Notification |
| `transfer.reversed.v1` | Transfer | Transaction, Notification |

The exact producer/consumer ownership must remain consistent with the service design. Commands and facts should be distinguished in event names and documentation.

## 3. Message key and ordering

Use `transferId` as key for transfer workflow events where per-transfer ordering is required. Ordering is guaranteed only within a Kafka partition, not globally across topics.

## 4. Example ledger posting command

```json
{
  "eventId": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
  "eventType": "LEDGER_POSTING_REQUESTED",
  "schemaVersion": 1,
  "occurredAt": "2026-11-23T10:00:00Z",
  "producer": "transfer-service",
  "aggregateType": "TRANSFER",
  "aggregateId": "33333333-3333-3333-3333-333333333333",
  "correlationId": "corr-example",
  "payload": {
    "transferId": "33333333-3333-3333-3333-333333333333",
    "postingReference": "POST-TRF-EXAMPLE",
    "sourceAccountId": "11111111-1111-1111-1111-111111111111",
    "destinationAccountId": "22222222-2222-2222-2222-222222222222",
    "amount": "5000.00",
    "currency": "INR"
  }
}
```

## 5. Producer requirements

- Use stable event IDs.
- Publish through an outbox for business state changes.
- Set message key consistently.
- Include schema version and correlation metadata.
- Configure producer acknowledgements and idempotence appropriately.
- Handle publish failure without losing the outbox record.

## 6. Consumer requirements

- Validate envelope and payload.
- Deduplicate by `(consumer_name, event_id)` or a domain-specific unique business reference.
- Persist consumer business effect and inbox marker atomically in the consumer database.
- Commit offset only after durable processing outcome.
- Retry transient errors with bounded backoff.
- Route poison/permanent failures to DLQ with metadata.
- Make replay safe.

## 7. Retry and DLQ

- Distinguish transient infrastructure errors from permanent validation/business errors.
- Define retry count and backoff per topic.
- Preserve original payload, headers, key, topic, partition, offset and failure reason in DLQ metadata.
- DLQ replay requires authorization, audit, validation and duplicate protection.
- Never replay a financial command blindly.

## 8. Topic configuration checklist

For each environment, document:
- Partitions
- Replication factor
- `min.insync.replicas`
- Retention
- Cleanup policy
- Consumer group
- Retry strategy
- DLQ topic
- Alert thresholds

Local single-broker development settings are not production recommendations.
