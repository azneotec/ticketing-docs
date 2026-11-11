# Architecture Decision Records

An ADR records one significant decision: the context, what we chose, what we rejected and what it costs us. New ADRs start from [ADR-000-template.md](ADR-000-template.md).

## Rules
- File name: `ADR-NNN-short-title.md`, numbered in order. The planned numbers are in the ADR backlog in `../plan/2026-11-08-project-plan.md`.
- **Accepted ADRs are never edited, only superseded.** To change a decision, write a new ADR that says "Supersedes ADR-NNN", and change the old one's status to `Superseded by ADR-MMM`. Fixing a typo is fine; changing the meaning is not.
- Keep each one short enough to explain aloud in about 60 seconds.
- Reasons come from the decision log (`../plan/2026-11-09-decision-log.md`) or from measurements, never invented afterwards.

## Index
| ADR | Title | Status | Date |
|---|---|---|---|
| [001](ADR-001-adr-format-and-location.md) | ADR format and location | Accepted | 2026-11-09 |
| [002](ADR-002-event-ticketing-domain.md) | Event ticketing as the domain | Accepted | 2026-11-09 |
| [003](ADR-003-repo-per-service.md) | One Git repo per service (polyrepo) | Accepted | 2026-11-09 |
| [004](ADR-004-strict-hexagonal-archunit.md) | Strict hexagonal architecture, enforced by the build | Accepted | 2026-11-10 |
| [005](ADR-005-java-25-features.md) | Java 25 features used deliberately (records, sealed types, pattern matching, virtual threads) | Accepted | 2026-11-10 |
| [006](ADR-006-jsonb-over-mongodb.md) | Postgres JSONB instead of MongoDB for seat maps | Accepted | 2026-11-10 |
| [007](ADR-007-observability-day-one.md) | Observability from day one (OpenTelemetry, `grafana/otel-lgtm` locally) | Accepted | 2026-11-10 |
| [010](ADR-010-rest-edge-grpc-inside.md) | REST at the edge, gRPC inside | Accepted | 2026-11-10 |
| [016](ADR-016-schema-registry-compatibility.md) | Schema registry and compatibility rules (Apicurio, BACKWARD, build time + runtime) | Accepted | 2026-11-10 |
| [019](ADR-019-module-federation-split.md) | Micro-frontends with Module Federation: shell + Discover, Checkout, Account | Accepted | 2026-11-12 |
| [027](ADR-027-constrained-hardware.md) | Running a distributed system on constrained hardware (16 GB laptop) | Accepted | 2026-11-10 |
| [028](ADR-028-shared-postgres-database-per-service.md) | One shared Postgres container, one database per service | Accepted | 2026-11-10 |
| [029](ADR-029-spotless-google-java-format.md) | Formatting enforced by the build (Spotless + google-java-format) | Accepted | 2026-11-10 |
| [030](ADR-030-avro-events-protobuf-grpc.md) | Avro for events, Protobuf for gRPC | Accepted | 2026-11-10 |
| [031](ADR-031-spring-data-jpa-separate-entities.md) | Spring Data JPA in the persistence adapter, entities separate from the domain | Accepted | 2026-11-10 |
| [032](ADR-032-problem-details-and-etag-if-match.md) | REST errors as Problem Details; optimistic locking via ETag / If-Match | Accepted | 2026-11-11 |
| [033](ADR-033-localhost-only-admin-api.md) | Localhost-only admin API until Phase 2 | Accepted (temporary) | 2026-11-11 |
| [034](ADR-034-grpc-server-and-opentelemetry-wiring.md) | gRPC server on Spring Boot's built-in support; OpenTelemetry through Micrometer, no agent | Accepted | 2026-11-11 |
| [035](ADR-035-frontend-tooling.md) | Front-end tooling: Rsbuild, Tailwind, TanStack Query, generated API types, Vitest/MSW/Playwright | Accepted | 2026-11-12 |
| [036](ADR-036-browser-tracing.md) | Browser tracing with W3C trace context, exported through a same-origin OTLP route | Accepted | 2026-11-12 |
