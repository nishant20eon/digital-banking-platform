# Digital Banking Platform

A learning/reference implementation of an enterprise-style digital banking backend built with Java 25, Spring Boot 4.2, Spring Security, Spring Data JPA, PostgreSQL, Apache Kafka, Redis, Docker, JUnit 5 and Mockito.

> **Important:** This repository is an engineering learning project, not a production-certified banking system. Do not use real customer data, real money, production credentials or live payment rails. A real deployment requires jurisdiction-specific legal, regulatory, security, operational and compliance review.

## Project goals

Build a service-oriented banking platform that supports:

- Customer registration, authentication and profile management
- Bank account creation and account lifecycle management
- Beneficiaries
- Internal account-to-account transfers
- Double-entry ledger and transaction history
- Idempotent APIs and duplicate event protection
- Kafka-based asynchronous communication
- Transactional outbox, retries and dead-letter queues
- Saga orchestration and compensating actions
- Reconciliation, audit, observability and admin operations

## Technology baseline

| Area | Choice |
|---|---|
| Language | Java 25 LTS |
| Framework | Spring Boot 4.2.0 (managed in root `pom.xml`) |
| Cloud | Spring Cloud 2026.0.0 |
| Security | Spring Security, JWT |
| Persistence | Spring Data JPA / Hibernate |
| Database | PostgreSQL 18.7 |
| Messaging | Apache Kafka 4.4.0 (single-node KRaft locally) |
| Cache / rate limiting | Redis 8.2 (extended-support release) |
| Gateway | Spring Cloud Gateway |
| Discovery | Eureka (optional local discovery component) |
| Resilience | Resilience4j |
| Migrations | Flyway |
| Tests | JUnit 5, Mockito, Testcontainers |
| API docs | OpenAPI |
| Containers | Docker / Docker Compose |
| Observability | Micrometer, Prometheus, Grafana, OpenTelemetry |

## Repository map

```text
digital-banking-platform/
├── README.md
├── pom.xml
├── .gitignore
├── .env.example
├── docker-compose.yml
├── docs/
│   ├── 00-project-charter.md
│   ├── 01-product-requirements.md
│   ├── 02-architecture.md
│   ├── 03-domain-model-and-state-machines.md
│   ├── 04-database-design.md
│   ├── 05-api-contracts.md
│   ├── 06-kafka-event-contracts.md
│   ├── 07-security-and-non-functional-requirements.md
│   ├── 08-testing-strategy.md
│   ├── 09-sprint-plan.md
│   ├── 10-backlog-epics-and-user-stories.md
│   ├── 11-definition-of-done.md
│   ├── 12-local-development-and-git-workflow.md
│   ├── 13-technology-stack-and-version-policy.md
│   └── adr/
│       ├── 0001-microservices-and-database-ownership.md
│       ├── 0002-ledger-and-money-representation.md
│       ├── 0003-saga-and-outbox.md
│       └── 0004-idempotency-and-concurrency.md
├── infrastructure/
│   └── README.md
└── services/
    ├── api-gateway/
    ├── discovery-server/
    ├── user-service/
    ├── account-service/
    ├── beneficiary-service/
    ├── transfer-service/
    ├── ledger-service/
    ├── transaction-service/
    ├── notification-service/
    └── admin-service/
```

## Start here

1. Read [Project Charter](docs/00-project-charter.md).
2. Read [Architecture](docs/02-architecture.md) and the ADRs.
3. Review [Product Requirements](docs/01-product-requirements.md).
4. Follow [Sprint Plan](docs/09-sprint-plan.md), starting with Sprint 0/1.
5. Implement stories from [Backlog](docs/10-backlog-epics-and-user-stories.md).
6. Use [Definition of Done](docs/11-definition-of-done.md) for every story.

## Local infrastructure

Prerequisites: Docker Desktop or Docker Engine, JDK 25, Maven 3.9.8 or later, Git.

```bash
cp .env.example .env
docker compose up -d
docker compose ps
```

The initial Compose file starts PostgreSQL 18.7, Kafka 4.4.0 (single-node KRaft for development) and Redis 8.2. PostgreSQL 18 uses `/var/lib/postgresql` as the Docker volume mount path. It is not a production topology. Application services are added and run as their own Maven modules in later sprints.

Default local-only ports:
- PostgreSQL: `5432`
- Kafka: `9092`
- Redis: `6379`

Never commit `.env`. Change local credentials before sharing your environment.

## Engineering rules

- A service owns its schema and data. No cross-service table access.
- Use `BigDecimal` in Java and `NUMERIC` in PostgreSQL for money. Never use `float`/`double`.
- A ledger posting is immutable after commit; correct it with compensating entries.
- Do not treat Redis as the source of truth for account balances.
- Do not assume Kafka makes PostgreSQL + Kafka + multiple services one atomic transaction.
- Make commands and consumers idempotent.
- For uncertain outcomes, query by a stable business reference before retrying or compensating.
- Keep business state transitions explicit and validated.
- Use synthetic test data only.

## Scope boundary

The first release supports **internal transfers between accounts in the same platform and same currency**. External bank rails, cards, FX, lending, real payment-provider integration, KYC/AML production integrations and regulatory certification are future scope and require separate design.
