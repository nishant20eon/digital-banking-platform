# 13 – Technology Stack and Version Policy

**Baseline selected:** 23 November 2026. Versions are pinned deliberately; do not use floating `latest` tags for reproducible builds.

## Approved baseline

| Component | Version / policy | Notes |
|---|---|---|
| Java | 25 LTS | Project SDK, Maven runtime and compiler release must all use JDK 25. |
| Spring Boot | 4.2.0 | Managed centrally through the Spring Boot dependency BOM. |
| Spring Cloud | 2026.0.0 | Release train aligned with Spring Boot 4.2.0. |
| Maven | 3.9.8 or later | Existing 3.9.8 installation is sufficient; Maven 3.10.x can be adopted separately. |
| Maven Compiler Plugin | 3.16.0 | Uses the `maven.compiler.release` property. |
| Maven Surefire Plugin | 3.6.0 | Unit-test execution. |
| PostgreSQL | 18.7 | Docker image `postgres:18.7`; mount the named volume at `/var/lib/postgresql`. |
| Apache Kafka | 4.4.0 | Docker image `apache/kafka:4.4.0`; local single-node KRaft. |
| Redis | 8.2 | Docker image `redis:8.2`; selected as an extended-support release rather than a short-lived newest minor. |

## Dependency management rules

1. Keep Spring Boot and Spring Cloud versions in the root `pom.xml` only.
2. Import the Spring Boot and Spring Cloud BOMs in the root POM.
3. Do not specify versions for Spring-managed starters, Spring Security, Spring Data, Spring Kafka, Jackson or Micrometer in child modules unless an explicit compatibility reason is documented.
4. Do not force the Kafka Java client to match the broker version. Use the version managed by Spring Boot unless compatibility testing proves an override is necessary. The broker and client are independently versioned.
5. Pin Docker image tags. Avoid `latest` for local reproducibility.
6. Before upgrading a major framework or database version, check its official compatibility matrix, migration guide and release notes, then run the complete test suite.
7. Record intentional deviations in an Architecture Decision Record under `docs/adr/`.

## Java configuration consistency

Use JDK 25 in all of these places:

- IntelliJ Project SDK
- IntelliJ Maven importer JDK
- IntelliJ Maven runner JRE
- Windows `JAVA_HOME` and terminal `PATH`
- CI build JDK
- Docker build/runtime image when application Dockerfiles are introduced

Verify from the repository root:

```powershell
java -version
mvn -version
```

Both commands must report Java 25. A successful Maven build with no source files only proves that the Maven reactor is configured; it does not yet prove that the Spring Boot services compile or run.

## PostgreSQL 18 Docker volume note

The official PostgreSQL Docker image changed its data directory layout in PostgreSQL 18. Mount the named volume at `/var/lib/postgresql`, not `/var/lib/postgresql/data`. If an older PostgreSQL container has already initialized the existing `postgres_data` volume, do not reuse it blindly with PostgreSQL 18. Back up any data you need and follow a proper database migration. For a disposable local database only, the volume can be recreated after confirming there is no data to preserve.

## Why Redis 8.2 instead of the newest Redis minor?

Redis 8.2 is selected because it is an extended-support release. A newer standard-support minor may exist, but the project prioritizes a longer maintenance window for its baseline.
