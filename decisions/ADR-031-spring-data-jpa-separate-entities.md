# ADR-031: Spring Data JPA in the persistence adapter, entities separate from the domain

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning round 13; built in session 006)

## Context
Catalogue needs a persistence adapter for Postgres. The domain is framework-free (ADR-004), so it can't carry `@Entity`. Azad wants to close real gaps in JPA knowledge (persistence context, dirty checking, lazy loading, locking), and JPA/Hibernate is what most Java teams run, so it matters in interviews. JPA hides a lot, so the adapter must make its behaviour visible rather than "magic".

## Decision
- **Spring Data JPA on Hibernate 7** (Spring Boot 4.1.2 → Hibernate 7.4) in `adapter-persistence`. Flyway owns the schema; Hibernate runs with `ddl-auto=validate`.
- **JPA entities (`EventEntity`, `VenueEntity`) live only in the adapter.** They are package-private, mutable, and shaped like the tables. An explicit `EventMapper` converts entity ↔ domain. Adapters implement the application's ports (`EventRepository`, `VenueRepository`) and translate errors (e.g. a JPA optimistic-lock failure becomes `ConcurrentUpdateException`).
- **Optimistic locking:** `@Version` on `EventEntity`. The domain `Event` carries the version it was loaded with as an opaque token. `save` compares it with the stored version *before* copying state onto the managed entity, because the managed entity always has the current version, and Hibernate's own `where version = ?` check only covers the gap between that load and the flush.
- **Short transactions in the adapter** (`@Transactional` on adapter methods). Use cases do read → change → save across two transactions, and the version token makes that safe.
- **Lists use a projection.** `ListEvents` is one JPQL constructor query (event joined with venue, no seat map) plus a count, so there is no N+1. `@ManyToOne(fetch = LAZY)` on the venue.
- `open-in-view=false`: no lazy loading outside the adapter's transactions.

## Alternatives considered
- **Annotate the domain classes with JPA** — less code, but it breaks ADR-004 and lets the ORM dictate the domain (no-arg constructors, mutable fields, no records, proxies).
- **jOOQ or Spring Data JDBC** — more explicit SQL and no persistence context. Good options, but they don't teach JPA, which was the reason for this choice (planning round 13).
- **Plain JDBC (`JdbcClient`)** — most transparent and most code to write. We still use it in tests to inspect raw rows.
- **MapStruct for mapping** — removes hand-written mappers but adds an annotation processor and hides the conversion. With a few aggregates, hand-written code is clearer.

## Consequences
**Good**
- The domain stays pure; the table shape and the domain shape can evolve separately (and the JSONB shape is pinned by its own records, ADR-006).
- JPA's behaviour is visible and tested: the SQL for save/load was read and explained (handoff 006), `NPlusOneTest` counts statements, `OptimisticLockingTest` shows Hibernate's version check, and stale domain copies are rejected.
- Swapping persistence technology touches one module.

**Bad (costs we accept)**
- **Mapping code**: entity classes, mappers and their tests for every aggregate, roughly doubling persistence code.
- Updates load the entity first (one SELECT before every INSERT/UPDATE), so dirty checking has something to compare against.
- JPA pitfalls still apply inside the adapter and need care. Example found in session 006: a package-private `getId()` on `VenueEntity` couldn't be intercepted by Hibernate's lazy proxy, so every event load also loaded its venue. Making it `public` fixed it; the SQL log showed the problem.
- Hibernate's default UPDATE writes every column (including the seat-map JSON) even if one changed. `@DynamicUpdate` would change that; not needed yet.

## Interview one-liner
"I used Spring Data JPA, but kept the entities inside the persistence adapter with explicit mappers, so the domain stays pure. The cost is mapping code. The aggregate carries its version as an optimistic-lock token, so read and write can be separate short transactions, and lists use projections to avoid N+1."
