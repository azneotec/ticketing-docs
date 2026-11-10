# ADR-016: Schema registry and compatibility rules (Apicurio, BACKWARD, build time + runtime)

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning rounds 4 and 11; build-time check built in session 004)

## Context
Kafka events outlive the code that wrote them: they stay in topics, are replayed, and are read by services that upgrade at different times (polyrepo, ADR-003). A producer that changes a schema carelessly breaks consumers it does not know about, often only in production. We need rules for how schemas may change, and something that enforces them before a bad schema is used.

## Decision
- **Compatibility rule: BACKWARD.** A consumer using the new schema (reader) must be able to read events written with the previous one (writer). In practice: new fields need defaults, enums need a default symbol, fields are not removed without a default on the reader side. This fits events because consumers are upgraded first and must still read events already sitting in Kafka.
- **Build time (now):** `checkAvroCompatibility` in `ticketing-contracts` runs Avro's `SchemaValidator` (can-read strategy = BACKWARD) against the same schema in the last release tag, plus a lint (namespace/path, docs, defaults). It runs in `check`, so an incompatible schema is never released. Protobuf has the same with `buf breaking`.
- **Runtime (Phase 3): Apicurio Registry** with a global `BACKWARD` rule. Producers register schemas on first use; the registry rejects incompatible ones, and each message carries a schema id so consumers fetch the writer schema.
- **Apicurio over Confluent Schema Registry** for licence reasons: Apicurio is Apache 2.0; Confluent's registry is under the Confluent Community License (source-available, restricted). Apicurio offers a Confluent-compatible API.

## Alternatives considered
- **Runtime registry only** — catches breaks only when a service starts producing, i.e. after merge and deploy; the build-time check gives the same answer on the pull request.
- **Build-time check only** — cannot see schemas registered by other producers, and does not give consumers the writer schema at runtime; Avro needs the writer schema to decode.
- **FORWARD or FULL compatibility** — FORWARD (old consumers read new data) suits "producers upgrade first"; FULL requires both and is the strictest. BACKWARD is the common default and matches our upgrade order; FULL can be set per subject later if needed.
- **Confluent Schema Registry** — the most common choice, but the licence restricts some uses.

## Consequences
**Good**
- Breaking schema changes fail the build, before anything is deployed.
- Consumers can always read old events, which keeps replays and slow upgrades safe.
- The runtime registry later adds a second, independent guard and schema ids on the wire.

**Bad (costs we accept)**
- BACKWARD alone does not protect old consumers from new data; producers must still add fields with defaults and avoid new enum symbols that old readers cannot map (hence enum defaults).
- Two places define "compatible" (build check and registry rule); they must stay the same.
- One more service to run in Phase 3, and Apicurio is less widely documented than Confluent.

## Interview one-liner
"Event schemas are checked twice for BACKWARD compatibility: in the build against the last released schema, so a breaking change never ships, and at runtime by Apicurio Registry, which also gives consumers the writer schema — Apicurio because it's Apache 2.0, unlike Confluent's registry."
