# ADR-004: Strict hexagonal architecture, enforced by the build

- **Status:** Accepted
- **Date:** 2026-11-10 (hexagonal decided in planning rounds 1 and 4; enforcement built in session 003)

## Context
Inside each service we use strict hexagonal architecture (ports and adapters): a pure, framework-free domain at the centre, use cases around it, and adapters (web, persistence, Kafka, gRPC) at the edge (planning round 1). Round 4 made it explicit: the domain has **no framework dependencies**. Over a 12–15 month project, with many fresh agent sessions writing code, a rule that lives only in a document will drift. Then one `@Entity` or `@Component` in the domain quietly ties the core to a framework. The rule has to be checked by the build, the same way in every one of about ten service repos.

## Decision
We enforce the architecture in two independent layers, shipped from `ticketing-build-conventions` (v0.1.0):
1. **Gradle domain dependency guard** (`ticketing.domain-conventions`): the domain module's main compile and runtime classpaths may contain **no external library at all** unless it is on an explicit allow-list (`domainDependencyGuard.allowed`). It runs before `compileJava`, so the build fails with the name of the offending dependency.
2. **ArchUnit rules library** (`ticketing-arch-rules`): every service has a one-line test, `HexagonalArchitecture.forBasePackage("...").check()`, that checks the compiled classes:
   - the domain depends only on itself and the JDK
   - no Spring, Jakarta EE or JPA in the domain
   - the application depends only on itself and the domain
   - domain and application never depend on adapters
   - adapters do not depend on each other

Package convention: `<base>.domain`, `<base>.application`, `<base>.adapter.<name>`. The domain rule fails if it finds no classes, which catches a mistyped base package.

## Alternatives considered
- **Documentation and code review only** — not enough here. Fresh agent sessions have no memory, and nobody reviews every line.
- **ArchUnit only, no Gradle guard** — ArchUnit sees only classes that are actually used. A framework dependency sitting unused on the domain classpath passes, and a violation shows up as a test failure late in the build. The Gradle guard fails earlier and names the dependency.
- **Gradle guard only, no ArchUnit** — sees modules, not packages. It cannot catch an adapter calling another adapter, or Spring code placed in the `domain` package of the Spring module (case 4 in `samples/verify-failure-cases.sh` proves this gap).
- **A deny-list (ban only `org.springframework`, `jakarta.persistence`, …)** — any framework not on the list (Jackson, Micronaut, Kafka clients) would get through. The allow-list is stricter and still lets us add a small framework-free library on purpose, with a reason.
- **Java Platform Module System (`module-info.java`)** — enforces module boundaries at compile time, but adds friction with Spring and test tooling, and does not express rules on layers inside a module.
- **Rules copied into every repo, or `buildSrc` per repo** — they would drift between ~10 repos. One versioned conventions repo keeps every service on the same rules and makes upgrades a version bump (polyrepo, ADR-003).

## Consequences
**Good**
- A framework leak in the domain fails `./gradlew build` locally, before review. `samples/verify-failure-cases.sh` proves it.
- The same rules, at the same version, in every service; a rule change is a release of one repo.
- The rules are executable documentation of the architecture.

**Bad (costs we accept)**
- Mapping code at the edges: persistence and web need their own models plus mappers, because domain classes can't carry JPA or Jackson annotations.
- Every domain library needs a deliberate allow-list entry. Even small helpers (e.g. JSpecify annotations) take a conscious decision.
- The ArchUnit test must live in a module whose test classpath sees all layers (the Spring Boot module), and services must follow the package convention.
- Services consume a versioned plugin. Until it is published to a real repository, that means `mavenLocal` and a manual `publishToMavenLocal`.

## Interview one-liner
"The domain is plain Java, and the build proves it: a Gradle guard fails if any non-allow-listed library reaches the domain classpath, and a shared ArchUnit test checks that dependencies only point inward. Two layers, because each catches what the other can't."
