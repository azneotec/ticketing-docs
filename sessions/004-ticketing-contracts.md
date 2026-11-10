# Session 004 — `ticketing-contracts` (Protobuf for gRPC, Avro for Kafka)
Phase: 1 · Estimated: 5–6 hours (split into 004a / 004b if needed; handoff at the split)

## Goal
A contracts repo that publishes versioned Java artifacts for gRPC (Protobuf) and Kafka events (Avro), where a breaking change to any published contract fails the build.

## Context to read
- `AGENTS.md`
- `ticketing-docs/plan/2026-11-08-project-plan.md`, `ticketing-docs/plan/2026-11-09-decision-log.md`
- `ticketing-docs/sessions/003-build-conventions-handoff.md` (conventions usage, versioning, known issues)

## Decisions already made (record them; do not re-decide)
- gRPC contracts: **Protobuf**, checked with **Buf** (`buf lint` + `buf breaking`).
- Kafka events: **Avro** (chosen to learn the mature schema-registry ecosystem). Schema files as `.avsc` (JSON), namespace per bounded context, e.g. `io.github.azneotec.ticketing.catalogue.v1`.
- Avro compatibility: **build-time** check with Avro's `SchemaValidator` against the last released schemas (Buf-like), **runtime** check later with **Apicurio Registry** (Apache 2.0) using a `BACKWARD` rule (Phase 3).
- Compatibility default: **BACKWARD** (new consumers can read old data) — explain why this suits event consumers.

## Part A — housekeeping
1. `ticketing-docs`: add ADR-029 to the plan's ADR backlog; append **Round 11** to the decision log with the decisions above (Avro over Protobuf/JSON for events, Buf, Avro build-time + Apicurio runtime checks, Apicurio over Confluent for licence reasons, `gh repo create` stays a manual step for Azad).
2. `ticketing-build-conventions` → release **0.1.1**: exclude generated sources (`build/generated/**`) from Spotless and JaCoCo; bump the sample to `0.1.2-SNAPSHOT` per README "Releasing". Tag `v0.1.1` (Azad pushes the tag if blocked).

## Part B — `ticketing-contracts` repo
3. `git init` on `master`, `.editorconfig`, README, Gradle wrapper 9.9.0, apply `ticketing.java-conventions` 0.1.1 from mavenLocal.
4. Layout:
   ```
   proto/ticketing/catalogue/v1/catalogue_service.proto
   avro/catalogue/v1/EventPublished.avsc
   buf.yaml, buf.gen? (only if useful)
   ```
5. **Protobuf / gRPC module** (`contracts-grpc`): `protobuf` Gradle plugin generating Java messages + gRPC stubs. First contract `CatalogueService` v1: `GetEvent`, `ListEvents` (paged), messages for event, venue summary, schedule (use `google.protobuf.Timestamp`, explicit time zone field per the time-zone note in the plan). Explain field numbers and `reserved`.
6. **Buf**: `brew install bufbuild/buf/buf` (Azad approves the install); `buf.yaml` with `STANDARD` lint and `FILE`-level breaking rules; a Gradle task (or script wired into `check`) running `buf lint` and `buf breaking --against` the last release tag. First release has nothing to compare against — handle that cleanly.
7. **Avro module** (`contracts-avro`): Avro Gradle plugin generating Java `SpecificRecord` classes. First schema `EventPublished` v1 (eventId, title, venueId, startsAt as `timestamp-millis` logical type + zone id, status enum with a **default** symbol — explain why enums need defaults for evolution).
8. **Avro compatibility task** `checkAvroCompatibility` in `check`: load each `.avsc`, load the same schema from the previous release (from the git tag or the previous published jar), run `SchemaValidatorBuilder().canReadStrategy().validateLatest()` (BACKWARD). Also a light lint: namespace pattern, `doc` on every record and field, defaults on new fields.
9. Prove it (script like `samples/verify-failure-cases.sh`): baseline passes; removing a proto field without `reserved` fails `buf breaking`; renaming a proto field type fails; adding an Avro field **without default** fails; adding one **with default** passes.
10. Publish `ticketing-contracts-grpc` and `ticketing-contracts-avro` to mavenLocal; tag `v0.1.0`.
11. ADRs in `ticketing-docs/decisions/` (update index; add numbers to the plan backlog):
    - **ADR-010** REST at the edge, gRPC inside.
    - **ADR-016** schema registry and compatibility rules (Apicurio, BACKWARD, build-time + runtime).
    - **ADR-030** Avro for events, Protobuf for gRPC (two IDLs: why it's worth it, what it costs).
12. Conventional Commits. **Azad runs** `gh repo create azneotec/ticketing-contracts --public --source . --push` and pushes tags; give him the exact commands.

## Acceptance criteria
- `./gradlew build publishToMavenLocal` passes in `ticketing-contracts`; failure script shows all cases behave as expected.
- Generated code is not formatted or coverage-checked (conventions 0.1.1).
- ADR-010, 016, 030 written and indexed; decision log Round 11; plan backlog updated.
- All three repos clean and pushed; tags `v0.1.1` (conventions) and `v0.1.0` (contracts) on GitHub.

## Out of scope
- Running Apicurio or Kafka (Phase 2/3), Schema Registry serializers in services.
- Any service implementation of the gRPC API.
- Pact.

## Concepts to explain (2–4 lines each)
IDL and code generation · Protobuf field numbers and `reserved` · package versioning (`v1`) · backward vs forward vs full compatibility · Avro schema resolution (writer vs reader schema) · why Avro needs defaults and enum default symbols · logical types · what a schema registry does at runtime and why it complements build-time checks · Buf lint vs breaking.
