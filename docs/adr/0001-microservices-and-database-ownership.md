# ADR-0001: Microservices and database ownership

- Status: Accepted for learning project
- Date: 2026-11-23

## Context
The project is intended to demonstrate independently deployable service boundaries and event-driven integration. Shared database access would create tight coupling and obscure ownership.

## Decision
Use domain-oriented Spring Boot services. Each service owns its persistence and exposes APIs/events. A single local PostgreSQL container may host separate logical databases for convenience, but services must not query one another's tables.

## Consequences
- More deployment/configuration complexity than a modular monolith.
- Eventual consistency must be designed explicitly.
- Cross-service queries require APIs or projections.
- Local development remains manageable with Docker Compose.

## Revisit when
The service count creates unnecessary operational overhead. A modular monolith may be a better first implementation if the learning objective changes.
