# ADR-030: Avro for events, Protobuf for gRPC

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning round 11; built in session 004)

## Context
We have two kinds of contracts: synchronous gRPC calls (ADR-010) and asynchronous Kafka events (ADR-016). gRPC fixes the IDL: it uses Protobuf. For events we could reuse Protobuf, or choose another format. The plan's goal is learning the tools behind real distributed systems, and the schema-registry ecosystem grew up around Avro.

## Decision
- **gRPC contracts: Protobuf** (`proto/`, artifact `ticketing-contracts-grpc`), checked with Buf.
- **Kafka events: Avro** (`avro/*.avsc`, artifact `ticketing-contracts-avro`), Java `SpecificRecord` classes generated with Avro's own compiler; namespace per bounded context and version, e.g. `io.github.azneotec.ticketing.catalogue.v1`.
- Both live in one repo (`ticketing-contracts`), share one version, and the build fails on incompatible changes for both.
- Times in both: a UTC instant (`google.protobuf.Timestamp` / `timestamp-millis`) plus an IANA zone id.

## Alternatives considered
- **Protobuf for events too** — one IDL and one toolchain, and registries support it. Rejected to learn Avro's model (schema resolution with writer and reader schemas, no field numbers), which is what most Kafka material and registries assume.
- **JSON Schema for events** — human-readable messages, but larger, slower, and weaker evolution rules; compatibility checking of JSON Schema is less mature.
- **Plain JSON without a schema** — no contract at all; breaking changes are found by consumers in production.

## Consequences
**Good**
- Each format is used where it is strongest: Protobuf with gRPC; Avro with Kafka and a registry (compact binary, schema travels by id).
- We learn both evolution models: Protobuf's field numbers and `reserved`, Avro's schema resolution and defaults.

**Bad (costs we accept)**
- Two IDLs, two code generators, two sets of evolution rules and checks to maintain.
- A concept that appears in both (e.g. event status) is defined twice and can drift; the Catalogue maps between them.
- Avro needs extra setup at runtime: the writer schema (from the registry) and a trusted-packages setting (`org.apache.avro.SERIALIZABLE_PACKAGES`).

## Interview one-liner
"Protobuf for gRPC because gRPC requires it, Avro for Kafka events because the schema-registry ecosystem is built around Avro's writer/reader schema resolution — the cost is two IDLs and two sets of evolution rules, which our contracts build checks automatically."
