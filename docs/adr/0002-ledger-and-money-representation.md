# ADR-0002: Ledger and money representation

- Status: Accepted
- Date: 2026-11-23

## Context
Financial amounts require exact decimal representation and auditable posting history.

## Decision
Use Java `BigDecimal` and PostgreSQL `NUMERIC`. Ledger postings are immutable after commit. Each journal balances by currency. Corrections use compensating entries referencing the original posting.

## Consequences
- Rounding and currency scale rules must be explicit.
- Journal balancing is enforced in domain logic and transaction boundary.
- Ledger history is append-only from application perspective.
- A reversal is a new posting, not an update/delete.

## Revisit when
Multi-currency, fees, FX, settlement accounts or a more sophisticated accounting model are introduced.
