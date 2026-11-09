# Handoff 003 — Housekeeping + `ticketing-build-conventions`
Date: 2026-11-10

## Done
Completed in one session (no 003a/003b split).

**Part A — `ticketing-docs` (pushed)**
- Decision log **Round 10** added (domain A/B/D reasons, monorepo, docs home A vs B, ArchUnit library, Spotless, GitHub email/public, agent rules versioned).
- ADR-001..003: alternatives completed with reasons (`docs: complete alternatives in ADR-001..003`).
- `AGENTS.md` moved to `ticketing-docs/AGENTS.md`; top-level `AGENTS.md` is now a relative symlink `AGENTS.md -> ticketing-docs/AGENTS.md`; `CLAUDE.md` unchanged (`@AGENTS.md`). Verified: `cat AGENTS.md` works, and a fresh `claude -p` session in the top-level folder quoted a rule from it.

**Part B — `ticketing-build-conventions` (new repo, tagged `v0.1.0`)**
- Gradle wrapper 9.9.0, Kotlin DSL, `gradle.properties` (`-Xmx1g`, caching), `.editorconfig`, `.gitattributes`, README.
- `plugins/` included build with three precompiled script plugins, published as `io.github.azneotec.ticketing:ticketing-gradle-plugins` plus plugin markers:
  - `ticketing.java-conventions`:
    - Java 25 toolchain; `-parameters -Xlint:all -Werror` (no lint relaxed so far)
    - JUnit 6.1.3 (Jupiter) + AssertJ
    - JaCoCo 0.8.16, 80% line-coverage gate in `check`. It also runs when a module has code but no tests, so no data counts as 0% and fails.
    - Spotless 8.11.0 + google-java-format 2.0.0, with `spotlessCheck` in `check`
  - `ticketing.domain-conventions`: `checkDomainDependencies` runs before `compileJava` and in `check`. It fails on **any** external library on the domain's compile or runtime classpath unless listed in `domainDependencyGuard.allowed`.
  - `ticketing.spring-boot-conventions`: Spring Boot **4.1.2** plugin and BOM (4.2.0 is still an RC), `bootRun -Xmx256m`; `spring.threads.virtual.enabled=true` documented as required.
- `arch-rules/` → `io.github.azneotec.ticketing:ticketing-arch-rules`:
  - `HexagonalArchitecture.forBasePackage("...").check()`, five rules, ArchUnit 1.5.1
  - 8 tests, 99% line coverage
- Version from Git tags (`gradle/version.gradle.kts`): exact tag → `X.Y.Z`, after a tag → next patch `-SNAPSHOT`, no tag → `0.1.0-SNAPSHOT`. Release steps are in the README.
- `samples/hexagonal-sample` (domain / application / Spring Boot adapter) consumes `0.1.0` from mavenLocal. `samples/verify-failure-cases.sh` result: **5 passed, 0 failed**:
  - baseline builds
  - Spring dependency + import in domain → guard fails
  - untested code → coverage < 80% fails
  - bad formatting → `spotlessCheck` fails
  - bonus: Spring class in the `domain` package of the Spring module → ArchUnit fails
- `./gradlew build publishToMavenLocal` passes; `v0.1.0` tagged locally.

## Done by Azad (after the build)
- **GitHub:** creating the public repo was blocked by Claude Code's auto-mode permission check, so Azad ran `gh repo create ... --push` and `git push origin v0.1.0` manually. Verified: https://github.com/azneotec/ticketing-build-conventions is public, `master` is in sync, and tag `v0.1.0` is on the remote.
- **IntelliJ check (step 12):** Azad confirmed that the per-project Gradle settings are Build and run = Gradle, Run tests = Gradle, Gradle JVM = Project SDK 25.

## Decisions made (and ADRs written)
- **ADR-004** strict hexagonal, enforced by the build (Gradle allow-list guard + ArchUnit). Indexed.
- **ADR-029** Spotless + google-java-format: a **separate short ADR** (not inside ADR-004), so each decision explains itself in 60 s. Indexed. *The plan's ADR backlog ends at 028; the elephant may want to add 029 there.*
- **Group ID** `io.github.azneotec.ticketing` (matches the GitHub account; valid for Maven Central later). Plugin ids `ticketing.*` as in the brief.
- **Allow-list, not deny-list**, for the domain guard (stricter; see ADR-004).
- **Repositories are not declared by the plugins**: each build declares them once in `settings.gradle.kts` (`dependencyResolutionManagement`). If a plugin declared them, they would silently override the settings ones (Gradle's default `PREFER_PROJECT` mode).
- **Root build does not forward `clean`** to the plugins build: doing that in one invocation deadlocks, because `arch-rules` is configured with those plugins. Full clean = `./gradlew clean && ./gradlew -p plugins clean`. Found while tagging; the unpushed tag was moved to the fix.
- ADR-001 also had a sentence saying the agent rules are "not versioned", which became untrue in this session. It was corrected in the same commit as the alternatives, and the `ticketing-docs` README line about `AGENTS.md` was updated.
- ArchUnit package convention: `<base>.domain`, `<base>.application`, `<base>.adapter.<name>`. The domain rule fails on "no classes" (catches a typo in the base package); the application and adapter rules allow empty, for early services.

## Files created or modified
Top-level folder (not a repo):
- `AGENTS.md` → now a symlink to `ticketing-docs/AGENTS.md`

`ticketing-docs/` (pushed):
- `AGENTS.md` (moved here; one line added saying where it lives)
- `README.md` (agent-rules line)
- `plan/2026-11-09-decision-log.md` (Round 10)
- `decisions/ADR-001-adr-format-and-location.md`, `ADR-002-event-ticketing-domain.md`, `ADR-003-repo-per-service.md` (alternatives)
- `decisions/ADR-004-strict-hexagonal-archunit.md` (new), `decisions/ADR-029-spotless-google-java-format.md` (new), `decisions/README.md` (index)
- `sessions/003-build-conventions.md` (brief, committed), `sessions/003-build-conventions-handoff.md` (this file)

`ticketing-build-conventions/` (new repo, pushed with tag `v0.1.0`):
- `.editorconfig`, `.gitattributes`, `.gitignore`, `gradle.properties`, `settings.gradle.kts`, `build.gradle.kts`, `README.md`
- `gradlew`, `gradlew.bat`, `gradle/wrapper/*`, `gradle/version.gradle.kts`
- `plugins/settings.gradle.kts`, `plugins/build.gradle.kts`
- `plugins/src/main/kotlin/`: `TicketingVersions.kt`, `CheckDomainDependencies.kt`, `ticketing.java-conventions.gradle.kts`, `ticketing.domain-conventions.gradle.kts`, `ticketing.spring-boot-conventions.gradle.kts`
- `arch-rules/build.gradle.kts`, `arch-rules/src/main/java/io/github/azneotec/ticketing/archrules/HexagonalArchitecture.java`
- `arch-rules/src/test/java/...`: `HexagonalArchitectureTest.java`, fixture classes under `archrules/fixtures/{good,framework,library,appleak,outward,sideways}/`, `org/springframework/fake/Component.java` (test stand-in)
- `samples/verify-failure-cases.sh`
- `samples/hexagonal-sample/`: `settings.gradle.kts`, `gradle.properties`, `{domain,application,adapter}/build.gradle.kts`, domain `Seat`, `SeatAlreadyHeldException`, `SeatTest`; application `SeatRepository`, `HoldSeatService`, `HoldSeatServiceTest`; adapter `SampleApplication`, `InMemorySeatRepository`, `application.properties`, `SampleApplicationTest`, `ArchitectureTest`

Outside the project: `~/.m2/repository/io/github/azneotec/ticketing/**` and `~/.m2/repository/ticketing/**` (published artifacts, `0.1.0` and `0.1.0-SNAPSHOT`).

## Known issues / blockers
1. **mavenLocal is temporary**: every service build must list `mavenLocal()`, and every machine must run `publishToMavenLocal` first. CI will need GitHub Packages (out of scope here).
2. **The sample's `gradle.properties` says `0.1.0`**. After the next change in this repo, set it to `0.1.1-SNAPSHOT` (README "Releasing", step 6), or the sample will keep testing the old release.
3. Spring Boot's test JVM prints `Sharing is only supported for boot loader classes…` (CDS warning caused by the JaCoCo agent). Harmless.
4. google-java-format style differs from IntelliJ's default: install the *google-java-format* IntelliJ plugin, or run `./gradlew spotlessApply` before committing.
5. Configuration cache is not enabled (Gradle suggests it). Candidate for a later convention.
6. Claude Code's auto mode blocks `gh repo create --public`. Future briefs that create repos should expect Azad to run that one command (or add a permission rule).

## What I learned (for interview prep)
- **Convention plugins vs `buildSrc` vs copy-paste.** Copy-pasting build logic into ten repos drifts. `buildSrc` shares logic only *within one build*. A convention plugin is build logic packaged as a versioned artifact, so ten repos apply the same rules by id and upgrade by bumping one version.
- **Gradle wrapper.** `gradlew` plus `gradle/wrapper/gradle-wrapper.properties` pin the Gradle version in the repo and download it on first use. Everyone, CI and agents included, builds with exactly 9.9.0, whatever is installed globally.
- **Java toolchains.** `java { toolchain { languageVersion = 25 } }` tells Gradle which JDK to compile and test with, separately from the JDK running Gradle. It finds (or provisions) that JDK, so builds don't depend on `JAVA_HOME`.
- **BOM (bill of materials).** A POM that only lists tested-together versions of many libraries. Importing Spring Boot's BOM as a `platform(...)` means we write `spring-boot-starter` without a version, and all Spring libraries line up.
- **JaCoCo, line vs branch coverage.** JaCoCo instruments bytecode during tests and records what ran. *Line* coverage = share of executable lines run at least once. *Branch* coverage = share of `if`/`switch` outcomes taken, which is stricter: one test can cover a line `if (a && b)` but not all its branches. Our gate is 80% lines.
- **ArchUnit.** A library that loads compiled classes and lets you assert their dependencies in a JUnit test ("domain may only depend on domain and `java..`"). Architecture becomes a failing test instead of a wiki page.
- **mavenLocal and why it's temporary.** `~/.m2/repository` is a Maven repository on this laptop. `publishToMavenLocal` puts artifacts there so other local builds can use them. It works only on this machine, has no access control or immutability, and CI can't see it, so a real repository (GitHub Packages) will replace it.
- **Spotless and build-enforced formatting.** Spotless runs a formatter as a Gradle task. Because `spotlessCheck` is part of `check`, unformatted code fails the build for everyone, including AI agents that never open an IDE. IDE settings are per-machine and can't fail anything.
- **Symlinks.** A small file holding a path to another file. A relative link (`ticketing-docs/AGENTS.md`) survives moving the whole folder. Git stores the link, not a copy.
- **Composite (included) builds.** `includeBuild("plugins")` lets one build use another build's plugins directly from source, without publishing first. Its gotcha: a task in the main build can't clean the build it is being configured with (our `clean` deadlock).

## Suggested next step
Phase 1 step 4: `ticketing-contracts` (protobuf build + first Catalogue `.proto`, versioned artifact). It can apply `ticketing.java-conventions` 0.1.0 from mavenLocal.
