# Session 006 — `catalogue-service` part 1: domain and persistence
Phase: 1 · Estimated: 5–6 hours (split into 006a / 006b if needed; handoff at the split)

## Goal
The first real service: a framework-free Catalogue domain written with modern Java, and a Spring Data JPA persistence adapter storing seat maps as JSONB, all proven by tests against real Postgres.

## Context to read
- `AGENTS.md`
- `ticketing-docs/plan/2026-11-08-project-plan.md`, `ticketing-docs/plan/2026-11-09-decision-log.md`
- `ticketing-docs/decisions/ADR-004-*.md` (hexagonal rules), `ADR-028-*.md` (database per service)
- `ticketing-docs/sessions/005-ticketing-platform-handoff.md` (database login, ports)
- `ticketing-build-conventions/README.md` (plugins, ArchUnit usage, mavenLocal)

## Decisions already made (record them; do not re-decide)
- Persistence: **Spring Data JPA (Hibernate 7)** — chosen to close Azad's JPA knowledge gaps and for interview relevance. JPA entities live only in the persistence adapter, separate from domain objects, with explicit mapping.
- Admin API (session 007) is **unauthenticated and localhost-only until Phase 2** (Keycloak).
- Catalogue split into two sessions: 006 domain + persistence; 007 REST/OpenAPI, admin API, gRPC server, OpenTelemetry, gate check.

## Part A — housekeeping
1. Append **Round 13** to the decision log (decisions above). Commit with this brief.

## Part B — `catalogue-service` repo
2. `git init` on `master`, `.editorconfig`, `.gitignore`, README, Gradle wrapper 9.9.0, conventions **0.1.2** from mavenLocal, version catalog, same tag-based versioning (copy `gradle/version.gradle.kts`).
3. Modules and plugins:
   - `domain` → `ticketing.domain-conventions` (no external libraries)
   - `application` → `ticketing.java-conventions` (depends on `domain` only)
   - `adapter-persistence` → Spring Data JPA, Flyway, Postgres driver
   - `bootstrap` → `ticketing.spring-boot-conventions`, wires everything; `spring.threads.virtual.enabled=true`
   - ArchUnit test using `ticketing-arch-rules` with base package `io.github.azneotec.ticketing.catalogue`.
4. **Domain** (pure Java, use features deliberately and note each in ADR-005):
   - `Venue` (id, name, city, IANA `ZoneId`).
   - `SeatMap` → sections → rows → seats (value objects as **records**, validated in compact constructors).
   - `Event` aggregate: title, venue, `startsAt` (`Instant`) + zone, seat map, status.
   - Status lifecycle as a **sealed interface** (e.g. `Draft`, `Published`, `Cancelled` records); transitions with **pattern matching for switch**, illegal transitions rejected. Domain events as a sealed hierarchy (`EventPublished`, `EventCancelled`) collected on the aggregate for later outbox use.
   - Typed ids (records wrapping `UUID`).
   - Unit tests for every rule; aim well above 80%.
5. **Application**: ports (`EventRepository`, `VenueRepository` as outbound ports; use cases `CreateVenue`, `CreateEvent`, `PublishEvent`, `GetEvent`, `ListEvents` with paging) and services implementing them. Plain Java, tested with in-memory fakes.
6. **Persistence adapter**:
   - Flyway migrations `V1__venues.sql`, `V2__events.sql`; `seat_map` column `jsonb`; index useful queries (e.g. by status + `starts_at`).
   - JPA entities + Spring Data repositories; JSONB via `@JdbcTypeCode(SqlTypes.JSON)`; mapper between entities and domain (no domain annotations).
   - **Optimistic locking** with `@Version` on the event entity.
   - Adapters implement the application ports.
   - Testcontainers tests (Postgres **18.6-alpine**, same as the platform): round-trip an event with a seat map; query by status; a JSONB query (e.g. events with a section named X) to show what JSONB gives us; optimistic-lock conflict raises the expected error.
   - Show (and explain) the SQL Hibernate runs for one save and one load, so lazy/eager loading and dirty checking are understood, not magic. Avoid N+1 in `ListEvents`.
7. `bootstrap`: Spring Boot app that starts against the platform's `catalogue` database (credentials from env, defaults matching `.env.example`), runs Flyway, and stops cleanly. `-Xmx256m`. No HTTP endpoints yet beyond actuator health.
8. ADRs in `ticketing-docs/decisions/` (update index):
   - **ADR-005** Java 25 features used deliberately (records, sealed interfaces, pattern matching, virtual threads) — where and why.
   - **ADR-006** Postgres JSONB instead of MongoDB for seat maps.
   - **ADR-031** Spring Data JPA in the persistence adapter, entities separate from the domain (and the cost: mapping code).
9. Conventional Commits; **Azad runs** `gh repo create azneotec/catalogue-service --public --source . --push`.

## Acceptance criteria
- `./gradlew build` passes: ArchUnit, domain guard, Spotless, 80% coverage, Testcontainers tests.
- `bootstrap` starts against `make up PROFILES="core"` and Flyway creates the tables in the `catalogue` database.
- ADR-005, 006, 031 written and indexed; Round 13 recorded; repos clean and pushed.

## Out of scope
- REST, OpenAPI, admin API, gRPC server, OpenTelemetry (session 007).
- Kafka, outbox, events leaving the service (Phase 2).
- Web shell.

## Concepts to explain (2–4 lines each)
Aggregate, entity, value object · records and compact constructors · sealed interfaces + pattern matching for state machines · ports and adapters in practice · JPA persistence context, dirty checking, flush · lazy vs eager loading and the N+1 problem · optimistic vs pessimistic locking · Flyway migrations and why they're immutable · Testcontainers lifecycle · why domain objects and JPA entities are kept apart.
