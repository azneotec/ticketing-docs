# ADR-005: Java 25 features used deliberately

- **Status:** Accepted
- **Date:** 2026-11-10 (Java 25 decided in planning; first used in catalogue-service, session 006)

## Context
The plan fixes Java 25 and says new language features are used *deliberately*, each with a reason. That rules out both extremes: ignoring them (writing Java 8 in a Java 25 project), and using them only to show them off. The catalogue domain is the first real code, so it sets the pattern every later service will copy. The domain has a small state machine (draft → published → cancelled), many small immutable values (seat map, ids), and later services will block on I/O (JDBC, gRPC, Kafka) under a tight memory budget (ADR-027).

## Decision
We use these features where they make a rule explicit or remove a whole class of bug:

| Feature | Where | Why |
|---|---|---|
| **Records + compact constructors** | `Seat`, `Row`, `Section`, `SeatMap`, `VenueId`, `EventId`, the `EventStatus` states, domain events, application commands, page types, JSONB document records | Immutable value objects in a few lines. The compact constructor validates and normalises (strips, copies lists with `List.copyOf`) before the fields are set, so an invalid or mutable value can't exist |
| **Typed ids (records wrapping `UUID`)** | `EventId`, `VenueId` | The compiler stops an event id being passed as a venue id; a bare `UUID` can't do that |
| **Sealed interfaces/classes** | `EventStatus` (`Draft`, `Published`, `Cancelled`), `DomainEvent` (`EventPublished`, `EventCancelled`), `DomainException` (`InvalidValueException`, `IllegalTransitionException`) | Closed set of cases the compiler knows. Each state carries only its own data (a cancel reason exists only on `Cancelled`) |
| **Pattern matching for `switch` + unnamed `_`** | `Event.publish` / `cancel` transitions | Exhaustive, no `default`: adding a state breaks the build at every switch that must decide what it means. Illegal transitions are explicit `case … -> throw` lines |
| **Record patterns** | `EventMapper.copyToEntity` (`case Published(var publishedAt) -> …`) | Takes a state apart in the `case` itself, with no casts or getters |
| **Text blocks** | JPQL / native SQL in `EventJpaRepository` | Multi-line queries readable as SQL |
| **Virtual threads** | `spring.threads.virtual.enabled=true` in every Spring service; one test starts a virtual thread | Blocking I/O without a big platform-thread pool, so a small heap (`-Xmx256m`) still serves many concurrent requests |

Not used (yet), on purpose: `ScopedValue`, structured concurrency, stream gatherers. No use case needs them; we add them with an ADR update when one does.

## Alternatives considered
- **Lombok for value objects** — rejected: records cover the need with no annotation processor, and the domain must stay library-free (ADR-004).
- **Enum + `if` chains for the lifecycle** — an enum can't carry per-state data (`publishedAt`, `reason`) without nullable fields on the aggregate, and `if` chains aren't checked for completeness.
- **State pattern with polymorphic methods** (`status.publish()`) — spreads the transition rules across classes, and lets states build the next state themselves. One exhaustive switch in the aggregate reads as a transition table.
- **Reactive stack (WebFlux, R2DBC) for concurrency** — virtual threads give the same scalability for blocking code, keep plain imperative code and stack traces, and work with JPA/JDBC.

## Consequences
**Good**
- Most domain rules are visible in one place, and the compiler checks them: a new status, domain event or exception kind can't be forgotten in a `switch`.
- Value objects are short, immutable and validated, and equality comes for free.
- Small heaps stay viable under blocking I/O.

**Bad (costs we accept)**
- Records can't hide fields or add lazy state; entities that change (`Event`) stay classes.
- Sealed hierarchies in one unnamed module must share a package, which shapes the package layout (the exceptions all live in `domain`).
- Virtual threads pin on `synchronized` blocks in old libraries (much improved since JDK 24); watch for this in load tests (Phase 4).
- Jackson/Hibernate must handle records (they do), and contributors must know Java 21+ syntax.

## Interview one-liner
"I used Java 25 features where they turn a rule into something the compiler checks: records with compact constructors for always-valid value objects, a sealed status with exhaustive pattern-matching switches for the event lifecycle, so a new state breaks the build until every transition handles it, and virtual threads so blocking JDBC fits a 256 MB heap."
