# Handoff 004 — `ticketing-contracts` (Protobuf for gRPC, Avro for Kafka)
Date: 2026-11-10

## Done
Completed in one session (no 004a/004b split).

**Part A — housekeeping**
- `ticketing-docs` (pushed):
  - Plan ADR backlog now lists 029 (Spotless) and 030 (Avro/Protobuf).
  - Decision log **Round 11** covers:
    - Avro for events
    - Buf
    - build-time and runtime Avro checks
    - Apicurio over Confluent (licence)
    - BACKWARD as the default rule
    - two IDLs
    - `gh repo create` stays manual
    - time zones and Avro codegen (both decided with Azad in this session)
  - Brief 004 committed.
- `ticketing-build-conventions` **0.1.1** (pushed, tag `v0.1.1` on GitHub):
  - `ticketing.java-conventions` now excludes generated code (`build/generated/**`) from Spotless (`targetExclude`) and from JaCoCo. JaCoCo only sees class files, so the plugin reads the `package` line of each generated `.java` file and excludes the matching classes, nested `Foo$Bar` included.
  - The coverage gate now counts only hand-written code, so a module with only generated code skips it.
  - New failure-script case `generated-code-ignored`. Result: **6 passed, 0 failed**. I also confirmed the generated class is missing from the JaCoCo XML report.
  - Release steps followed. The sample is now on `0.1.2-SNAPSHOT`. A follow-up commit replaced the deprecated `by tasks.registering` in the script.

**Part B — `ticketing-contracts` (new repo, pushed by Azad, tag `v0.1.0`)**
- Build setup: Gradle wrapper 9.9.0, `ticketing.java-conventions` 0.1.1 from mavenLocal, version catalog `gradle/libs.versions.toml`, and the same Git-tag versioning as conventions.
- Versions: protobuf 4.36.2, gRPC 1.85.1, Avro 1.12.2, protobuf Gradle plugin 0.10.1.
- `proto/ticketing/catalogue/v1/catalogue_service.proto`:
  - `CatalogueService` with `GetEvent` and `ListEvents` (paged with `page_size`, `page_token` and `next_page_token`).
  - Messages `Event`, `VenueSummary` and `Schedule` (`Timestamp` plus `zone_id`).
  - Enum `EventStatus` (`_UNSPECIFIED = 0`).
  - Java package `io.github.azneotec.ticketing.catalogue.grpc.v1`.
- `buf.yaml` (v2): STANDARD lint, FILE breaking rules. Buf 1.75.0 installed with Homebrew.
- `avro/catalogue/v1/EventPublished.avsc`:
  - fields `eventId`, `title`, `venueId`, `startsAt` (`timestamp-millis`, generated as `java.time.Instant`), `zoneId`, `status`
  - `status` is an `EventStatus` enum with default `UNKNOWN`
  - namespace `io.github.azneotec.ticketing.catalogue.v1`
- `contracts-grpc`: protobuf plugin generates messages and gRPC stubs. In `check`:
  - `bufLint`
  - `bufBreaking`, which compares against the latest `v*` tag, or against `-Pticketing.compatibilityBaselineTag`
  - a test running a fake server and client in-process, plus an unknown-field test
- `contracts-avro`: the `generateAvroJava` task (`buildSrc`, calls `avro-compiler` directly). `checkAvroCompatibility` runs in `check`:
  - BACKWARD check with `SchemaValidatorBuilder().canReadStrategy().validateLatest()` against `git show <tag>:<path>`
  - fails if a published schema is deleted
  - lint: namespace and path, `doc` everywhere, enum defaults, defaults on new fields
  - tests: a round trip, and a newer writer's unknown enum symbol becoming `UNKNOWN`
- First release: with no tag, both checks print "nothing to compare against (first release)" and pass. After tagging they print `comparing … with tag v0.1.0`.
- `samples/verify-failure-cases.sh` result: **6 passed, 0 failed**:
  - baseline passes
  - proto field removed → fails
  - proto field removed **with** `reserved` → still fails (see Decisions)
  - proto field type changed → fails
  - Avro field without a default → fails
  - Avro field with a default → passes
- `./gradlew build publishToMavenLocal` passes. `ticketing-contracts-grpc` and `ticketing-contracts-avro` `0.1.0` are in `~/.m2`, and `v0.1.0` is tagged.
- Generated Protobuf, gRPC and Avro code compiles under `-Xlint:all -Werror` with **no lint relaxed**, so the risk from the plan didn't happen.
- ADR-010, ADR-016 and ADR-030 written and indexed.

## Done by Azad (after the build)
- **GitHub:** `gh repo create` is blocked by Claude Code's auto mode (handoff 003), so Azad ran `gh repo create azneotec/ticketing-contracts --public --source . --push` and `git push origin v0.1.0`. Verified: https://github.com/azneotec/ticketing-contracts is public, `master` is in sync, and tag `v0.1.0` is on the remote.

## Decisions made (and ADRs written)
- **ADR-010** REST at the edge, gRPC inside. **ADR-016** BACKWARD, Apicurio, build time + runtime. **ADR-030** Avro for events, Protobuf for gRPC. All indexed.
- **Time zones** (Azad approved): UTC instant plus an IANA `zone_id` / `zoneId` string. The brief mentioned a "time-zone note in the plan" that doesn't exist; this is now in Round 11.
- **Avro codegen** (Azad approved): a ~40-line `buildSrc` task calling `avro-compiler`, because the `davidmc24` plugin is archived.
- **Buf FILE rules mean a field cannot be removed, even with `reserved`**: FILE also protects generated code (`getCity()` would vanish), and `reserved` only makes removal wire-safe (the WIRE rules). The way to retire a field is `[deprecated = true]` (I checked that it passes `buf breaking`) and then removing it in `v2`. This is documented in the README and proven by failure case 2. *The elephant may want WIRE_JSON rules instead if FILE feels too strict.*
- **Java packages:** the proto Java package is `…catalogue.grpc.v1`, separate from the Avro namespace `…catalogue.v1`. That way the two jars never share a Java package (split packages break JPMS and confuse readers).
- **Build logic lives in `buildSrc`**, not the conventions plugin: Buf and Avro tasks are only used by this repo.
- **Baseline override** `-Pticketing.compatibilityBaselineTag=<tag>` lets the failure script compare against a temporary local tag, so nothing is pushed and it works before the first release.
- **Avro trusted packages:** Avro ≥ 1.11.4 refuses to load generated classes from untrusted packages. Tests set `org.apache.avro.SERIALIZABLE_PACKAGES=io.github.azneotec.ticketing`, and the README tells every consumer service to do the same.
- **ADR-010 reasons:** the decision log records only the decision (rounds 1 and 4), not why. I used the standard reasoning (browser/BFF/cookie fit vs typed generated clients). *Elephant: confirm it, or add the reasons to the log.*

## Files created or modified
`ticketing-docs/` (pushed):
- `plan/2026-11-08-project-plan.md` (ADR backlog: 029, 030)
- `plan/2026-11-09-decision-log.md` (Round 11)
- `decisions/ADR-010-rest-edge-grpc-inside.md`, `ADR-016-schema-registry-compatibility.md`, `ADR-030-avro-events-protobuf-grpc.md` (new), `decisions/README.md` (index)
- `sessions/004-ticketing-contracts.md` (brief, committed), `sessions/004-ticketing-contracts-handoff.md` (this file)

`ticketing-build-conventions/` (pushed, tag `v0.1.1`):
- `plugins/src/main/kotlin/ticketing.java-conventions.gradle.kts`
- `samples/verify-failure-cases.sh`
- `samples/hexagonal-sample/gradle.properties` (0.1.1, then 0.1.2-SNAPSHOT)

`ticketing-contracts/` (new repo, pushed, tag `v0.1.0`):
- `.editorconfig`, `.gitattributes`, `.gitignore`, `gradle.properties`, `settings.gradle.kts`, `build.gradle.kts`, `README.md`, `buf.yaml`
- `gradlew`, `gradlew.bat`, `gradle/wrapper/*`, `gradle/version.gradle.kts`, `gradle/libs.versions.toml`
- `proto/ticketing/catalogue/v1/catalogue_service.proto`, `avro/catalogue/v1/EventPublished.avsc`
- `buildSrc/settings.gradle.kts`, `buildSrc/build.gradle.kts`, `buildSrc/src/main/kotlin/{Git,BufTasks,GenerateAvroJava,CheckAvroCompatibility}.kt`
- `contracts-grpc/build.gradle.kts`, `contracts-grpc/src/test/java/io/github/azneotec/ticketing/catalogue/grpc/v1/CatalogueServiceContractTest.java`
- `contracts-avro/build.gradle.kts`, `contracts-avro/src/test/java/io/github/azneotec/ticketing/catalogue/v1/EventPublishedContractTest.java`
- `samples/verify-failure-cases.sh`

Outside the project:
- Homebrew `buf` 1.75.0
- `~/.m2/repository/io/github/azneotec/ticketing/`:
  - conventions `0.1.1`, `0.1.1-SNAPSHOT` and `0.1.2-SNAPSHOT`
  - `0.1.0` republished from the tagged commit (see issue 1)
  - `ticketing-contracts-{grpc,avro}/0.1.0`

## Known issues / blockers
1. **Version trap in `version.gradle.kts`** (conventions and contracts): when `HEAD` is exactly on a release tag, the version is the release version **even with uncommitted changes**. In this session, an uncommitted build of conventions overwrote `0.1.0` in `~/.m2`. I restored it by stashing my changes and republishing from the tag (the jar size matches the original). *Suggestion for a future brief:* return `-SNAPSHOT` when the working tree is dirty (`git status --porcelain`). For now it's noted in the contracts README.
2. **Conventions README "Releasing" step 6** should add: "then `./gradlew publishToMavenLocal`, or the sample can't resolve the new `-SNAPSHOT`". The failure script failed for exactly this reason until I published.
3. mavenLocal is still the only repository (handoff 003, issue 1). Services will need `mavenLocal()` and a local `publishToMavenLocal` of contracts too.
4. Buf must be installed on every machine, and later in CI. Gradle fails with a clear "install buf" message if it isn't.
5. Buf and Avro checks run on every build (no declared outputs, because they depend on Git tags). They're fast (~1 s).
6. The ADR-010 reasons aren't in the decision log yet (see Decisions).

## What I learned (for interview prep)
- **IDL and code generation.** An Interface Definition Language (`.proto`, `.avsc`) describes messages and APIs without depending on any programming language. A compiler (`protoc`, `avro-compiler`) generates Java classes from it, so both sides of a contract use the same types, and changing the contract shows up as a compile error, not a runtime surprise.
- **Protobuf field numbers and `reserved`.** On the wire a field is just *number + value*, and names aren't sent. Renumbering or reusing a number makes old data decode into the wrong field. `reserved 3; reserved "city";` blocks the number and name forever after a field is removed. Unknown numbers are skipped, which is why adding fields is safe.
- **Package versioning (`v1`).** The major version is part of the package (`ticketing.catalogue.v1`). Compatible changes stay in `v1`. A breaking change goes into a new `v2` package that runs alongside `v1` until every caller has moved.
- **Backward, forward and full compatibility.**
  - BACKWARD: new readers can read old data, so consumers upgrade first.
  - FORWARD: old readers can read new data, so producers upgrade first.
  - FULL: both.
  - BACKWARD suits event consumers, which must read events already in the topic.
- **Avro schema resolution.** Avro binary data has no field names or numbers, so it can only be decoded with the **writer's** schema. The reader then maps it onto its own **reader** schema by field name:
  - fields the reader doesn't know are skipped
  - fields the reader has but the writer didn't get the reader's default
- **Why Avro needs defaults and enum default symbols.** If the reader has a field the old writer didn't write, the default is the only possible value, so a new field without a default makes old events unreadable. If a newer writer sends an enum symbol the reader doesn't know, the reader maps it to the enum's `default`. Without one, decoding fails (shown in `EventPublishedContractTest`).
- **Logical types.** An annotation on a primitive type that gives it meaning. `{"type":"long","logicalType":"timestamp-millis"}` is a long on the wire but `java.time.Instant` in code. In Protobuf, the equivalent is the well-known type `google.protobuf.Timestamp`.
- **What a schema registry does at runtime.**
  - Producers register a schema and send its id (a few bytes) with each message instead of the whole schema.
  - Consumers fetch the writer schema by id.
  - The registry rejects schemas that break its compatibility rule.
  - It complements the build-time check: it sees every producer, including ones outside this repo, and is the source of writer schemas at runtime.
- **Buf lint vs Buf breaking.** `buf lint` checks one version of the files for style: naming, `_UNSPECIFIED` zero values, package and directory match. `buf breaking` compares two versions (working tree vs a Git tag) and reports incompatible changes. Rule sets go from the loosest (WIRE: bytes only), through WIRE_JSON and PACKAGE, to FILE (generated code too).
- **`buildSrc`.** A special folder Gradle compiles before the build and puts on its classpath. It's the place for build logic that only this repo uses. Shared logic belongs in a published convention plugin.
- **Avro class trust list.** Since 1.11.4, Avro only creates instances of classes from packages listed in `org.apache.avro.SERIALIZABLE_PACKAGES`, which stops crafted schemas from instantiating arbitrary classes. Every consumer must set it.

## Suggested next step
Phase 1 step 5: `ticketing-platform`.
- Compose with the shared Postgres, Kafka (KRaft, 512 MB heap) and `grafana/otel-lgtm`, with resource limits on every container.
- ADR-007, ADR-027, ADR-028.

Optionally, first fix the version trap (issue 1) and the README step (issue 2) in a conventions 0.1.2 patch.
