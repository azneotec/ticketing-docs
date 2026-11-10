# Handoff 006 — `catalogue-service` part 1: domain and persistence
Date: 2026-11-10

## Done
Completed in one session (no 006a/006b split).

**Part A — housekeeping**
- Decision log **Round 13** (JPA with separate entities; admin API unauthenticated + localhost-only until Phase 2; catalogue split 006/007). Committed and pushed with brief 006.

**Part B — `catalogue-service` (new repo, 6 commits on `master`, not on GitHub yet)**
- **Skeleton:**
  - `.editorconfig`, `.gitignore`, wrapper and `gradle/version.gradle.kts` copied from conventions
  - Gradle 9.9.0, conventions **0.1.2** from mavenLocal
  - `gradle/libs.versions.toml`: Spring-managed libraries carry no version, the Boot BOM pins them
  - README
- **Modules:**
  - `domain`: `domain-conventions`
  - `application`: `java-conventions`, depends on `domain` only
  - `adapter-persistence`: `spring-boot-conventions` for the Boot BOM only (boot jar off)
  - `bootstrap`: `spring-boot-conventions`, `main()`, wiring, ArchUnit test
- **Resolved versions:** Spring Boot 4.1.2, Hibernate 7.4.13, Flyway 12.4.0, Postgres driver 42.7.14, Testcontainers 2.0.5.
- **Domain** (pure Java, **100% line and branch coverage**, 26 tests):
  - Records validated in compact constructors: `VenueId`, `EventId`, `Seat`, `Row`, `Section`, `SeatMap`. Unique labels at each level, non-empty, immutable copies.
  - `Venue` and `Event` are entities, equal by id. `Event` is the aggregate.
  - Sealed `EventStatus` (`Draft`, `Published(publishedAt)`, `Cancelled(cancelledAt, reason)`) with a `Code` enum for filters and storage.
  - Transitions use exhaustive pattern-matching switches:
    - publish only a draft that starts in the future
    - cancel a draft or a published event
    - anything else throws `IllegalTransitionException`
  - Sealed `DomainEvent` (`EventPublished`, `EventCancelled`), collected on the aggregate and pulled with `pullDomainEvents()`. Cancelling a *draft* records no event, because nobody outside the organiser has seen it.
  - Sealed `DomainException` (`InvalidValueException`, `IllegalTransitionException`), ready for an exhaustive HTTP mapping in 007.
- **Application** (100% coverage, 11 tests, in-memory fakes):
  - Inbound ports: `CreateVenue`, `CreateEvent`, `PublishEvent`, `GetEvent`, `ListEvents` (paged, filtered by a set of status codes, soonest first).
  - Outbound ports: `EventRepository`, `VenueRepository`.
  - `VenueService` and `EventService` (with an injected `Clock`).
  - Supporting types: `Page`, `PageRequest` (size 1–100), the `EventSummary` read model, `NotFoundException`, `ConcurrentUpdateException`.
- **Persistence adapter** (98.5% coverage, 11 Testcontainers tests on `postgres:18.6-alpine`, 256 MB limit):
  - Migrations:
    - `V1__venues.sql`
    - `V2__events.sql`: `seat_map jsonb`, check constraints that mirror the state rules, `(status, starts_at)` index, FK index on `venue_id`, GIN `jsonb_path_ops` index on `seat_map`
  - Entities, Spring Data repositories, `EventMapper` and adapters:
    - `@JdbcTypeCode(SqlTypes.JSON)` stores a separate `SeatMapDocument` record tree
    - `@Version` on `EventEntity`
    - `ddl-auto=validate`
  - Tests:
    - round trip with a seat map
    - raw JSONB checks via `jsonb_typeof` / `#>>`
    - status changes bump the version
    - summaries by status with paging and the venue join
    - JSONB containment query (events that have a "Balcony" section)
    - stale domain copy rejected (`ConcurrentUpdateException`)
    - Hibernate's own version check with two real transactions (`ObjectOptimisticLockingFailureException`)
    - N+1 shown and avoided: naive loop **6 statements** for 5 events; `ListEvents` **2 statements** (page + count)
    - corrupt JSON fails domain validation
- **Bootstrap** (5 tests):
  - Spring Boot app in the base package; `UseCaseConfiguration` turns the plain-Java services into beans.
  - `application.properties`:
    - virtual threads on
    - datasource from env with `.env.example` defaults
    - Hikari pool 5
    - `open-in-view=false`
    - graceful shutdown
    - only `/actuator/health` exposed
  - Tests:
    - ArchUnit `HexagonalArchitecture.forBasePackage("io.github.azneotec.ticketing.catalogue")`
    - Flyway history is `1, 2`
    - use cases end to end
    - health is UP over HTTP
    - `main()` starts and stops cleanly, observed with `SpringApplication.withHook`
- **`./gradlew build` passes:** ArchUnit, domain guard, Spotless, `-Werror`, 80% coverage per module, all Testcontainers tests. 53 tests in total.
- **Against the platform:** `make up PROFILES="core"`, then `./gradlew :bootstrap:bootRun`:
  - Flyway: "Successfully applied 2 migrations … now at version v2"
  - Started in 3.4 s
  - `/actuator/health` returns `{"status":"UP"}`
  - `\dt` in `catalogue` shows `event`, `venue` and `flyway_schema_history`, owned by `catalogue`, with all four indexes
  - RSS ~207 MB with `-Xmx256m`
  - SIGTERM gives a graceful shutdown (Tomcat, then the EntityManagerFactory, then Hikari)
  - Platform stopped afterwards with `make down` (volumes kept, so the tables are still there)
- **ADRs:** ADR-005, ADR-006 and ADR-031 written and indexed; 031 added to the plan's ADR backlog.

### SQL Hibernate runs (from the test log, explained)
Save a **new** event, then load it:
```
select … from event where id=?                         -- adapter: does it exist, and which version? (empty)
insert into event (…, seat_map, …, version, …) values  -- persist(); seat_map bound as JSON, version 0
select … from event where id=?                         -- findById: ONE query, venue not loaded
```
- No `select … from venue`, even though the entity has a venue: `getReferenceById` gives a **proxy** that holds only the foreign key, and `@ManyToOne(fetch = LAZY)` means loading an event doesn't load its venue.
- Publishing (dirty checking + optimistic lock):
  ```
  select … from event where id=?
  update event set …, status=?, published_at=?, version=? where id=? and version=?   -- 1, 0
  ```
  There is no `save()` call for the update. The entity is *managed*, and at flush Hibernate compares it field by field with the snapshot it took at load. Something changed, so it sends the UPDATE.
- Venue save is `select` + `insert`: the entity has an assigned id and no `@Version`, so Spring Data can't tell it's new and calls `merge()`.
- **Bug found by reading the SQL:** at first every event load *also* ran `select … from venue where id=?`. The mapper calls `entity.getVenue().getId()`. The getter was package-private, so Hibernate's proxy class couldn't override it, and the call initialised the proxy. Making `VenueEntity.getId()` public removed the query. The comment in the code and ADR-031 record it.

## Done by Azad (after the build)
- [ ] `cd catalogue-service && gh repo create azneotec/catalogue-service --public --source . --push`
- [ ] Optional: tag `v0.1.0` once happy (no release was asked for in this brief).

## Decisions made (and ADRs written)
- **ADR-005** Java 25 features used deliberately: records/compact constructors, typed ids, sealed status/events/exceptions, exhaustive pattern matching and record patterns, text blocks, virtual threads; and what we deliberately don't use yet.
- **ADR-006** JSONB instead of MongoDB for seat maps (GIN `jsonb_path_ops`, domain validates the shape).
- **ADR-031** Spring Data JPA with entities separate from the domain (mapping cost, version token, projections, the proxy-getter pitfall).
- Smaller choices (in code comments, not ADR-worthy):
  - **`adapter-persistence` uses `spring-boot-conventions`** (boot jar off) instead of `java-conventions` + a hand-written BOM. That way the Boot version stays in exactly one place, the conventions. This changes the plan I wrote at the start of the session.
  - **The domain `Event` carries `version`** as an opaque token. Without it, read and save in separate transactions could overwrite each other silently.
  - **Each adapter method is its own transaction.** Use cases aren't `@Transactional`; the version token makes read → save safe. The outbox in Phase 2 will need save + outbox insert in one transaction, inside the adapter.
  - **Tests use `@SpringBootTest` with real commits, not `@DataJpaTest`.** Its rollback-per-test would hide flushes, commits and the lock race. Tables are truncated before each test.
  - **Singleton Testcontainers pattern**, not `@Container` per class, so the container matches Spring's cached test context.
  - **The JSONB query (`findIdsWithSection`) is on the adapter but not on a port.** No use case needs it yet; search comes later.
  - **Status in the DB as `varchar`** with check constraints, mapped with `@Enumerated(STRING)` of the domain `EventStatus.Code`.
  - **`server.port` defaults to 8080** (`PORT` env). Kafbat stays on 8089.

## Files created or modified
`ticketing-docs/` (two commits, both pushed):
- `plan/2026-11-09-decision-log.md` (Round 13)
- `plan/2026-11-08-project-plan.md` (ADR backlog: 031)
- `sessions/006-catalogue-domain-persistence.md` (brief, committed), `sessions/006-catalogue-domain-persistence-handoff.md` (this file)
- `decisions/ADR-005-java-25-features.md`, `decisions/ADR-006-jsonb-over-mongodb.md`, `decisions/ADR-031-spring-data-jpa-separate-entities.md` (new), `decisions/README.md` (index)

`catalogue-service/` (new repo, 6 commits on `master`):
- Root: `.editorconfig`, `.gitignore`, `README.md`, `build.gradle.kts`, `settings.gradle.kts`, `gradle.properties`, `gradlew`, `gradlew.bat`, `gradle/wrapper/*`, `gradle/version.gradle.kts`, `gradle/libs.versions.toml`
- `domain/build.gradle.kts`
- `domain/src/main/java/io/github/azneotec/ticketing/catalogue/domain/`:
  - `DomainException.java`, `InvalidValueException.java`, `IllegalTransitionException.java`, `Guard.java`
  - `venue/`: `Venue.java`, `VenueId.java`
  - `seatmap/`: `Seat.java`, `Row.java`, `Section.java`, `SeatMap.java`
  - `event/`: `Event.java`, `EventId.java`, `EventStatus.java`, `DomainEvent.java`, `EventPublished.java`, `EventCancelled.java`
- `domain/src/test/java/.../domain/`: `GuardTest.java`, `venue/VenueTest.java`, `seatmap/SeatMapTest.java`, `event/EventTest.java`
- `application/build.gradle.kts`
- `application/src/main/java/.../application/`:
  - `NotFoundException.java`, `ConcurrentUpdateException.java`, `Page.java`, `PageRequest.java`, `EventSummary.java`
  - `port/in/`: `CreateVenue.java`, `CreateEvent.java`, `PublishEvent.java`, `GetEvent.java`, `ListEvents.java`
  - `port/out/`: `EventRepository.java`, `VenueRepository.java`
  - `service/`: `VenueService.java`, `EventService.java`
- `application/src/test/java/.../application/`:
  - `ApplicationModelTest.java`
  - `service/`: `EventServiceTest.java`, `InMemoryEventRepository.java`, `InMemoryVenueRepository.java`
- `adapter-persistence/build.gradle.kts`
- `adapter-persistence/src/main/resources/db/migration/`: `V1__venues.sql`, `V2__events.sql`
- `adapter-persistence/src/main/java/.../adapter/persistence/`:
  - `VenueEntity.java`, `EventEntity.java`, `SeatMapDocument.java`, `EventSummaryRow.java`
  - `VenueJpaRepository.java`, `EventJpaRepository.java`
  - `EventMapper.java`, `JpaVenueRepository.java`, `JpaEventRepository.java`
- `adapter-persistence/src/test/resources/application.properties`
- `adapter-persistence/src/test/java/.../adapter/persistence/`:
  - `PersistenceTestApplication.java`, `PostgresTestBase.java`, `TestData.java`
  - `JpaEventRepositoryTest.java`, `OptimisticLockingTest.java`, `NPlusOneTest.java`, `MapperTest.java`
- `bootstrap/build.gradle.kts`
- `bootstrap/src/main/java/io/github/azneotec/ticketing/catalogue/`: `CatalogueApplication.java`, `UseCaseConfiguration.java`
- `bootstrap/src/main/resources/application.properties`
- `bootstrap/src/test/java/io/github/azneotec/ticketing/catalogue/`: `ArchitectureTest.java`, `CatalogueApplicationTest.java`

Outside the project:
- Tables `venue`, `event` and `flyway_schema_history` now exist in the platform's `catalogue` database (Docker volume `ticketing_postgres-data`).
- Gradle cache: Spring Boot 4.1.2, Hibernate, Flyway and Testcontainers 2.0.5 dependencies.
- Testcontainers containers are removed automatically when the tests finish.

## Known issues / blockers
1. **`catalogue-service` isn't on GitHub yet.** Azad runs `gh repo create` (auto mode blocks creating public repos).
2. Git warns that `gradlew.bat` CRLF will become LF. The other repos behave the same; a `.gitattributes` (`*.bat text eol=crlf`) across all repos would fix it. Suggestion only.
3. **Domain events are collected but not stored.** `publishEvent` saves the aggregate, and the recorded `EventPublished` is dropped. This is intended until the outbox (Phase 2, ADR-012). The comment in `EventService.publishEvent` marks the spot.
4. Two lines in `JpaEventRepository.save` aren't covered by tests. They translate a race between our version check and the flush. Hibernate's side of it is proven separately in `OptimisticLockingTest`.
5. Every UPDATE rewrites all columns, including the seat-map JSON (Hibernate's default static SQL). Fine at this size; `@DynamicUpdate` is the lever if it ever matters.
6. Ryuk, Testcontainers' reaper container, runs without a memory limit (small, and only during tests).
7. `bootRun` exits with code 1 when stopped with SIGTERM. That is Gradle reporting the killed JVM; the app itself shut down gracefully.
8. Still open from earlier handoffs: mavenLocal is the only repository; Javadoc warnings in conventions `arch-rules`.

## What I learned (for interview prep)
- **Aggregate, entity, value object.**
  - An *aggregate* is a cluster that changes as one unit through its root, which enforces the rules (`Event`).
  - An *entity* has an identity that survives changes, so equality is by id (`Event`, `Venue`).
  - A *value object* is defined only by its values, immutable and equal by value (`Seat`, `SeatMap`, the ids).
  - Aggregates reference each other by id only (`Event.venueId`).
- **Records and compact constructors.** A record gives final fields, accessors, `equals`/`hashCode`/`toString`. The compact constructor runs before the fields are set, so it can validate and *reassign* parameters (strip a name, `List.copyOf` a list). Then an invalid or mutable record can't exist.
- **Sealed interfaces and pattern matching for state machines.** A sealed type lists every subtype, so a `switch` over it is exhaustive without `default`. Add a state and every switch fails to compile until it decides what the new state means. Each state record carries only its own data.
- **Ports and adapters in practice.**
  - Inbound ports are the use-case interfaces callers use.
  - Outbound ports are what the core needs (repositories).
  - Adapters implement or call them.
  - The proof: the application is tested with `HashMap` fakes, and the JPA adapter passes the same contract against real Postgres.
- **JPA persistence context, dirty checking, flush.**
  - The persistence context is the per-transaction map of managed entities (first-level cache: the same id gives the same object).
  - Dirty checking: at flush, Hibernate compares each managed entity with its load-time snapshot and sends UPDATEs only for changed ones, with no `save()` needed.
  - Flush is when SQL is actually sent: before commit, before some queries, or explicitly. We flush explicitly so a lock conflict surfaces inside the adapter.
- **Lazy vs eager loading and N+1.**
  - LAZY puts a proxy in place and queries only when it's touched; EAGER loads together.
  - N+1: one query for N rows, then one query per row when a lazy association is touched in a loop. Measured: 6 statements for 5 events.
  - Fixes: a join or constructor projection (our choice, 2 statements including the count), `join fetch`, entity graphs, or batch fetching.
  - Pitfall: a proxy can only answer `getId()` without a query if it can override the method, so it must not be package-private.
- **Optimistic vs pessimistic locking.**
  - Optimistic: no lock while reading. The writer checks `where version = ?` and fails if someone got there first. Good when conflicts are rare and transactions short.
  - Pessimistic: `select … for update` takes a row lock and others wait. Good under heavy contention, such as seat holds in Phase 2, but it holds locks and risks deadlocks.
  - Across separate transactions (read now, save later) the version must travel with the data.
- **Flyway migrations and immutability.** Versioned SQL files run once, in order. Each one's checksum is stored in `flyway_schema_history`, and Flyway refuses to start if an applied file changed. Fixes are new migrations, so every environment ends up with the same schema by the same path.
- **Testcontainers lifecycle.** A real Docker container per test JVM (singleton pattern). `@ServiceConnection` hands its URL and credentials to Spring Boot. Ryuk removes containers even if the JVM crashes. Spring caches test contexts, so the container must outlive any single test class.
- **Why domain objects and JPA entities are kept apart.**
  - JPA needs no-arg constructors, mutable fields, proxies and table-shaped classes. The domain wants immutability, invariants and its own shape.
  - Keeping them apart keeps the domain pure and lets the schema evolve on its own.
  - The cost is mapping code, plus remembering that the managed entity always holds the *current* version.

## Suggested next step
Session 007, `catalogue-service` part 2:
- REST + OpenAPI over the existing inbound ports, mapping the sealed `DomainException`, `NotFoundException` (404) and `ConcurrentUpdateException` (409) exhaustively
- the localhost-only admin API (create venue/event, publish)
- the gRPC server from contracts 0.1.1
- OpenTelemetry to `localhost:4317/4318`
- the Phase 1 gate check

It needs only `make up PROFILES="core"`. Before you start, push `catalogue-service` to GitHub.
