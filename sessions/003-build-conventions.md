# Session 003 — Housekeeping + `ticketing-build-conventions`
Phase: 1 · Estimated: 4–5 hours (may split into 003a / 003b; if so, write a handoff at the split point)

## Goal
Fix the ADR gaps from session 002, version the agent rules, and create a shared Gradle build-conventions repo that every service will use.

## Context to read
- `AGENTS.md` (top level)
- `ticketing-docs/plan/2026-11-08-project-plan.md`
- `ticketing-docs/plan/2026-11-09-decision-log.md`
- `ticketing-docs/sessions/002-ticketing-docs-repo-handoff.md`
- `ticketing-docs/sessions/001-workstation-setup-handoff.md` (IntelliJ per-project Gradle checks)

## Part A — housekeeping in `ticketing-docs`
1. Append **Round 10** to the decision log with these elephant-supplied reasons (verbatim in meaning):
   - Domain options rejected: **A order fulfilment** — the textbook saga, reads generic in interviews; **B travel booking** — more participants and complexity than a beginner-style start needs; **D wallet** — strong for interviews but too close to Azad's day job (banking), so less new learning. **C ticketing** chosen: seat holds with expiry, contention and idempotency make it naturally distributed.
   - Monorepo rejected: polyrepo chosen for realism (mirrors enterprise teams owning and releasing services independently); its costs (contracts repo, platform repo, Pact Broker, multi-PR changes) are accepted deliberately as learning and interview material. A monorepo would be faster solo but hides versioning and contract problems.
   - Docs home: **A** everything in `ticketing-docs` (versioned) — chosen; **B** plan/sessions stay in unversioned top-level folder — rejected (no history or backup).
   - Shared ArchUnit rules library: included from session 003. Formatting: Spotless + google-java-format from the first line of code.
   - GitHub email verified; visible commit email accepted. All repos public.
2. Update ADR-001, ADR-002, ADR-003 with these alternatives and reasons (allowed now as a correction: they were written yesterday and nothing depends on them yet). Commit as `docs: complete alternatives in ADR-001..003`.
3. Version the agent rules: move top-level `AGENTS.md` to `ticketing-docs/AGENTS.md`, replace the top-level file with a **relative symlink** (`ln -s ticketing-docs/AGENTS.md AGENTS.md`). `CLAUDE.md` stays `@AGENTS.md`. Show the exact change and wait for approval. Verify both `cat AGENTS.md` and Claude Code still read the rules. Update the paths/wording inside `AGENTS.md` if needed.
4. Commit and push `ticketing-docs`, including this brief.

## Part B — `ticketing-build-conventions` repo
5. Create `ticketing-build-conventions/`, `git init` on `master`, `.editorconfig` (same as ticketing-docs), README, Gradle wrapper **9.9.0**, Kotlin DSL, `gradle.properties` with `org.gradle.jvmargs=-Xmx1g` and `org.gradle.caching=true`.
6. Convention plugins (precompiled script plugins in an included build, published as a plugin artifact):
   - `ticketing.java-conventions` — Java 25 toolchain; `-parameters`, `-Xlint:all`, warnings as errors (relax only with a stated reason); JUnit 5; JaCoCo report + **80% line coverage gate** wired into `check`; **Spotless with google-java-format** (`spotlessCheck` in `check`).
   - `ticketing.domain-conventions` — applies java-conventions; **fails the build** if any `org.springframework*`, `jakarta.persistence*` or other framework dependency appears on the domain module's classpath.
   - `ticketing.spring-boot-conventions` — applies java-conventions; Spring Boot plugin + BOM at the latest stable Spring Boot; default JVM args for `bootRun` (`-Xmx256m`); `spring.threads.virtual.enabled=true` documented as the default the services must set.
7. `ticketing-arch-rules` library (in the same repo, separate module/artifact): reusable ArchUnit rules for strict hexagonal layering — domain depends on nothing outside itself; application depends only on domain; adapters depend inward only; no Spring in domain. One-line usage from a service test.
8. Versioning: version from a Git tag set by hand; tag `v0.1.0` after everything passes. Document how to bump.
9. Publish plugins and `ticketing-arch-rules` to **mavenLocal**. Add a small `samples/hexagonal-sample` project in the repo that consumes them (domain / application / adapter modules) and proves: build passes; adding a Spring import to domain fails; dropping coverage below 80% fails; bad formatting fails `spotlessCheck`.
10. Write **ADR-004** (strict hexagonal + ArchUnit enforcement + domain dependency guard) in `ticketing-docs/decisions/` and update the ADR index. Also record the Spotless decision either inside ADR-004's consequences or as a short ADR-029 (your choice — say which in the handoff).
11. Conventional Commits; create public GitHub repo `ticketing-build-conventions`, push `master` and the tag.
12. Open the repo in IntelliJ with Azad and check the per-project Gradle settings from handoff 001: build/run = Gradle, tests = Gradle, Gradle JVM = Project SDK 25.

## Acceptance criteria
- ADR-001..003 include alternatives and reasons; decision log has Round 10; `AGENTS.md` lives in `ticketing-docs` and the top-level symlink works.
- `./gradlew build publishToMavenLocal` passes in `ticketing-build-conventions`; the sample shows all four failure cases fail as expected.
- `v0.1.0` tagged and pushed; both repos clean and pushed.
- ADR-004 written and indexed.

## Out of scope
- Contracts repo, platform repo, any real service.
- Publishing to GitHub Packages or CI (later).
- Automated versioning from Conventional Commits.

## Concepts to explain (2–4 lines each)
Convention plugins vs `buildSrc` vs copy-paste · Gradle wrapper · Java toolchains · BOM (bill of materials) · JaCoCo and what line vs branch coverage means · ArchUnit · `mavenLocal` and why it's a temporary step · Spotless and why formatting is enforced in the build, not the IDE · symlinks.
