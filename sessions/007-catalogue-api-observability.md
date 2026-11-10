# Session 007 — `catalogue-service` part 2: REST, admin API, gRPC, observability
Phase: 1 · Estimated: 5–6 hours (split into 007a = REST + admin, 007b = gRPC + observability + gate; handoff at the split)

## Goal
Expose the Catalogue use cases over REST (public + localhost-only admin) and gRPC, send traces/metrics/logs to the local Grafana stack, prove the first half of the Phase 1 gate, and release `catalogue-service` v0.1.0.

## Context to read
- `AGENTS.md`
- `ticketing-docs/plan/2026-11-08-project-plan.md`, `ticketing-docs/plan/2026-11-09-decision-log.md`
- `ticketing-docs/decisions/ADR-004`, `ADR-007`, `ADR-010`, `ADR-031`
- `ticketing-docs/sessions/006-catalogue-domain-persistence-handoff.md`
- `ticketing-contracts/README.md` (gRPC artifact, Avro trusted packages note)
- `ticketing-platform/README.md` (ports, OTLP endpoints)

## Decisions already made (record them; do not re-decide)
- Errors use **Problem Details (RFC 9457)**; every `DomainException` subtype + `NotFoundException` + `ConcurrentUpdateException` mapped by an exhaustive switch.
- Optimistic locking over HTTP with **`ETag` / `If-Match`** (409 on mismatch, 428 if `If-Match` missing on state-changing admin calls).
- Admin API `/admin/**` is unauthenticated until Phase 2 but **enforced localhost-only** by a filter (loopback remote address only; 403 otherwise). Keycloak roles replace it in Phase 2.
- OpenTelemetry via **Spring Boot 4's built-in support** (Micrometer tracing/OTLP), **no Java agent** (memory).
- gRPC server: **Spring gRPC if it supports Spring Boot 4.1**, otherwise plain grpc-java managed by a Spring lifecycle bean. Check, decide, record.
- Release `catalogue-service` **v0.1.0** at the end. Browser part of the gate is session 008.

## Part A — housekeeping
1. Append **Round 14** to the decision log (decisions above). Commit with this brief.
2. Add `.gitattributes` to `catalogue-service` (`*.bat text eol=crlf`, others LF), matching the other repos.

## Part B — REST + admin (007a)
3. New module `adapter-rest` (Spring MVC). Public endpoints:
   - `GET /events?status=PUBLISHED&page=0&size=20` → paged `EventSummary` list (default: published only, soonest first).
   - `GET /events/{id}` → event detail with seat map, `ETag` = version.
   Request/response DTOs are records local to the adapter; no domain types in JSON.
4. Admin endpoints:
   - `POST /admin/venues` → 201 + `Location`.
   - `POST /admin/events` → 201 + `Location` + `ETag`.
   - `POST /admin/events/{id}/publish` with `If-Match` → 200 + new `ETag`.
   Localhost-only filter with tests (simulate a non-loopback remote address → 403).
5. Problem Details `@RestControllerAdvice`: 400 invalid value, 404 not found, 409 illegal transition / version conflict, 412 or 409 for `If-Match` mismatch (pick one, justify), 428 missing `If-Match`. Validation errors list field problems.
6. **OpenAPI** via springdoc (Boot 4-compatible version); Swagger UI at `/swagger-ui.html` locally; commit the generated `openapi.yaml` into the repo (`docs/openapi.yaml`) via a Gradle task or test, so the web shell can read it.
7. **CORS** for the web shell dev origin (`http://localhost:5173` by default, configurable), public endpoints only.
8. Tests: `@WebMvcTest` slices for controllers + advice; one full `@SpringBootTest` + Testcontainers flow (create venue → create event → publish with If-Match → list → stale If-Match gets 409/412).

## Part C — gRPC + observability (007b)
9. New module `adapter-grpc` using `ticketing-contracts-grpc` **0.1.1** (mavenLocal): implement `GetEvent` and `ListEvents`; map errors to gRPC status (`NOT_FOUND`, `INVALID_ARGUMENT`, `FAILED_PRECONDITION`); honour client deadlines; port 9090 (configurable). Tests with an in-process server + client.
10. Observability:
    - Spans for HTTP, gRPC and JDBC (datasource observation), one trace per request across use case and SQL; add a custom span or observation around use cases if Boot doesn't give one.
    - Metrics and logs exported over OTLP to `localhost:4318` (or 4317); trace id and span id in every log line.
    - Service name `catalogue-service`, resource attributes (version, environment=local).
    - Keep RSS near the 256 MB heap budget; report measured RSS.
11. **Gate check part 1** (with `make up PROFILES="core"`): `curl` one REST read and one admin flow, and `grpcurl` one `GetEvent`. In Grafana Tempo show a single trace containing HTTP → use case → SQL spans, and one for gRPC; logs for that trace in Loki; one metric (e.g. HTTP request count) in Prometheus. Provide links/queries; **Azad confirms by eye**.

## Part D — wrap-up
12. README: how to run, endpoints table, admin/localhost note, gRPC usage with `grpcurl`, observability links.
13. ADRs in `ticketing-docs/decisions/` (update index, add numbers to the plan backlog):
    - **ADR-032** REST errors as Problem Details; optimistic locking via ETag/If-Match.
    - **ADR-033** localhost-only admin API until Phase 2 (risk, enforcement, replacement).
    - Record the gRPC server choice (Spring gRPC vs plain grpc-java) and the OTel approach inside ADR-007's follow-up or a short **ADR-034** — your choice; say which.
14. `./gradlew build` green; tag **v0.1.0** following the conventions release steps; push (`git push` and tag push work from Claude Code; ask Azad only if blocked).

## Acceptance criteria
- REST, admin and gRPC endpoints work and are tested; build passes all gates (ArchUnit, Spotless, 80% coverage, Testcontainers).
- Non-loopback admin calls get 403; stale `If-Match` is rejected.
- `docs/openapi.yaml` committed; CORS allows the shell origin.
- Traces (REST and gRPC), logs and a metric visible in Grafana; Azad confirms.
- ADRs written and indexed; Round 14 recorded; `catalogue-service` v0.1.0 tagged and pushed; all repos clean.

## Out of scope
- Web shell (session 008), gateway/BFF, Keycloak, Kafka/outbox, Avro events leaving the service.
- Event cancellation endpoint (domain supports it; expose in Phase 4 with refunds).

## Concepts to explain (2–4 lines each)
REST resource design and status codes · Problem Details · ETag/If-Match and conditional requests (412 vs 428) · CORS and preflight · OpenAPI and why commit the spec · gRPC status codes and deadlines · in-process gRPC testing · traces, spans, context propagation · Micrometer Observation vs OpenTelemetry API · log/trace correlation · RED metrics (rate, errors, duration).
