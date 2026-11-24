# 12 – Local Development and Git Workflow

## 1. Prerequisites

- Git
- JDK 25 (use the same JDK for IntelliJ Project SDK, Maven importer, Maven runner and terminal)
- Maven 3.9.8 or later
- Docker Desktop / Docker Engine and Compose plugin
- IDE such as IntelliJ IDEA
- Postman or Bruno (optional)

## 2. Clone and start dependencies

```bash
git clone <your-repository-url>
cd digital-banking-platform
cp .env.example .env
docker compose up -d
docker compose ps
```

The local Compose baseline is PostgreSQL 18.7, Apache Kafka 4.4.0 in KRaft mode, and Redis 8.2. PostgreSQL 18 stores its data under `/var/lib/postgresql`; the Compose volume is mounted at that parent directory.

Check dependencies:
```bash
docker compose logs -f postgres
docker compose logs -f kafka
docker compose logs -f redis
```

## 3. Local ports

| Component | Port |
|---|---:|
| PostgreSQL | 5432 |
| Kafka external listener | 9092 |
| Redis | 6379 |
| Eureka (when app module is implemented) | 8761 |
| API Gateway (when app module is implemented) | 8080 |

## 4. Suggested branch strategy

- `main`: stable, reviewed code.
- `develop`: integration branch if using GitFlow.
- `feature/US-101-customer-registration`
- `bugfix/<short-description>`
- `docs/<short-description>`

For a solo project, trunk-based development with short-lived feature branches is also suitable.

## 5. Commit examples

```text
feat(user): implement customer registration
test(user): add duplicate email validation tests
feat(account): add atomic fund reservation
docs(architecture): document transfer saga
fix(ledger): reject duplicate posting reference
```

## 6. Suggested implementation sequence

For each story:
1. Read the epic and acceptance criteria.
2. Identify owning service.
3. Write migration first.
4. Define DTO/API or event contract.
5. Implement domain logic.
6. Add unit tests.
7. Add integration tests.
8. Update OpenAPI/event docs.
9. Run full build and tests.
10. Open a PR and record design decisions.

## 7. Java and Maven verification

Run these commands from the repository root and confirm both commands report Java 25:

```powershell
java -version
mvn -version
```

If IntelliJ shows JDK 25 but `mvn -version` reports another Java version, update `JAVA_HOME` and the Windows `Path`, then restart the terminal/IDE.

## 8. Local safety

- Use fake customers and account numbers.
- Do not use real passwords, OTPs or personal data.
- `.env` must never be committed.
- Do not connect local test code to a real bank/payment provider.
- Reset local data only when you understand the effect on your test workflow.
