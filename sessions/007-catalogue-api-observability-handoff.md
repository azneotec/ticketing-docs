# Handoff 007 — `catalogue-service` part 2: REST, admin API, gRPC, observability
Date: 2026-11-11

Done in one sitting, with the interim handoff [`007a-catalogue-rest-admin-handoff.md`](007a-catalogue-rest-admin-handoff.md) at the split. That file has the full REST detail; this one covers everything, with 007b in detail.

## Done
**Part A — housekeeping:** decision log Round 14; `.gitattributes` in `catalogue-service`.

**Part B — REST + admin (007a):**
- New module `adapter-rest`:
  - Public `GET /events`, `GET /events/{id}`.
  - Admin create venue/event, get venue/event, and publish with `If-Match`.
- Problem Details for every error (400/403/404/409/412/428).
- Localhost-only admin guard.
- CORS for the web shell.
- springdoc 3.2.0, with `docs/openapi.yaml` committed and checked by a test.
- `PublishEvent` takes the caller's expected version; new `GetVenue` use case.
- See handoff 007a for the endpoint table and the choices made.

**Part C — gRPC + observability (007b):**
- **gRPC server choice:** Spring Boot 4.1 includes Spring gRPC (`spring-boot-starter-grpc-server`, Spring gRPC 1.1.2). Used instead of plain grpc-java.
- **New module `adapter-grpc`** (`CatalogueGrpcService`), implementing `ticketing.catalogue.v1` from `ticketing-contracts-grpc` 0.1.1:
  - `GetEvent`: any status, including the venue name and city (a second use case call, `GetVenue`).
  - `ListEvents`: published only, soonest first. `page_size` 0 means 20, capped at 100, negative is rejected. `page_token` is opaque base64url.
  - **Status mapping** (`GrpcErrors`, with an exhaustive inner switch over the sealed `DomainException`):
    - `NOT_FOUND`, `INVALID_ARGUMENT`, `FAILED_PRECONDITION`.
    - `ABORTED` for version conflicts.
    - `INTERNAL` for anything else, with the message withheld and the error logged.
  - **Deadlines:** an expired call does no work, and a cancelled call sends no reply.
  - Port 9090 (`GRPC_PORT`), server reflection and the `grpc.health.v1` health service on.
  - `grpc-java` aligned to **1.85.1** with `grpc-bom`. The Boot BOM says 1.84.1; the contracts were built with 1.85.1.
  - **Tests:**
    - In-process server and client (real serialisation, codes and deadlines).
    - A test that drives the gRPC `Context` by hand for the exact expiry and cancel cases.
    - In `bootstrap`, a network test (`GrpcApiTest`) against Postgres, including the health check.
    - Adapter coverage 100%.
- **Observability** (no Java agent):
  - `spring-boot-starter-opentelemetry`: Micrometer Observation → OTel SDK → OTLP/HTTP (`OTEL_EXPORTER_OTLP_ENDPOINT`, default `http://localhost:4318`). 100% sampling locally; metrics pushed every 15 s.
  - **Spans:**
    - HTTP and gRPC from Boot.
    - JDBC from `datasource-micrometer` 2.3.0 (`connection`, `query`).
    - Use cases from `ObservedUseCases`, a JDK proxy around each inbound port registered as a `@Primary` bean in `UseCaseConfiguration`. It also produces the `catalogue.use-case` timer (RED per use case).
  - **Logs:** `[traceId-spanId]` in every console line inside a request. The OTel Logback appender (`logback-spring.xml` + `OpenTelemetryLogging`) sends them to Loki. Admin changes now log at INFO (audit trail, and the gate needed a log line on a traced request).
  - **Resource:** `service.name=catalogue-service`, `service.version` (Gradle version substituted into `application.properties`), `deployment.environment.name=local`.
  - **`TracingTest`** proves one HTTP request = one trace (server span → `GetEvent` → SQL spans, same trace id) and that the use-case timer counted it. Other integration tests switch OTLP export off.
- **Gate check part 1** (`make up PROFILES="core"` + `bootRun`). Confirmed by Azad in Grafana.
  - REST read and admin flow with `curl`: 201 (`ETag "0"`) → 428 without `If-Match` → 200 (`ETag "1"`) → 412 on stale `"0"` → public read and list 200.
  - `grpcurl list` and `GetEvent` OK.
  - **Tempo:** REST trace `http post /admin/events/{id}/publish` → `PublishEvent` → `connection`/`query`. gRPC trace `ticketing.catalogue.v1.CatalogueService/GetEvent` → `GetEvent`, `GetVenue` → SQL.
  - **Loki:** `{service_name="catalogue-service"} | trace_id="…"` returns `Published event … (version 0 -> 1)`.
  - **Prometheus:** `sum by (method, uri, status) (http_server_requests_milliseconds_count{service_name="catalogue-service"})` shows the 201/428/200/412 calls. `grpc_server_*` and `catalogue_use_case_*` are present.
  - **Measured:** start-up ~4.5 s. RSS ~95 MB idle, **~280 MB after 30 requests** with `-Xmx256m` (session 006 measured ~207 MB; the extra is Netty for gRPC and the OTel SDK). Platform stopped afterwards with `make down` (volumes kept, so the gate data is still in the `catalogue` database).

**Part D — wrap-up:**
- README rewritten: modules, run, env vars, REST table, error table, admin note, `curl` flow, gRPC + `grpcurl`, error codes, deadlines, Grafana queries, release steps.
- **ADR-032** (Problem Details, ETag/If-Match), **ADR-033** (localhost-only admin), **ADR-034** (gRPC server choice and OTel wiring). I chose a separate ADR-034 rather than a follow-up inside ADR-007: ADR-007 is the principle; 034 is how Spring wires it, and it will be the template for the next services. Indexed; 032–034 added to the plan's ADR backlog.
- `./gradlew build` green: **119 tests**. Coverage: domain 100, application 100, persistence 98.5, rest 96.1, grpc 100, bootstrap 100.
- **`catalogue-service` v0.1.0 tagged and pushed** (tag on `b7d7d98`; the build on the tag reports `version: 0.1.0`; the jar's `catalogue.version=0.1.0`).

## Decisions made (and ADRs written)
- ADR-032, ADR-033, ADR-034 (above). Summary of the smaller choices:
  - 412 for a stale `If-Match`, 409 for illegal transitions. `If-Match: *` → 428, malformed → 400.
  - Admin guard is an MVC **interceptor**, not a servlet filter (same path matching as the controllers). The round 14 intent is unchanged.
  - Public REST hides drafts. gRPC `GetEvent` returns every status (trusted callers). gRPC `ListEvents` returns published events only.
  - `ABORTED` for version conflicts in gRPC (Google's guidance for concurrency conflicts).
  - Use-case observation by proxy in `bootstrap` rather than `@Observed`, which would put Micrometer and AspectJ into the application module.
  - Logback appender pinned to **2.28.0-alpha**, the instrumentation release built on OTel 1.62, which Boot 4.1.2 manages.

## Files created or modified
`ticketing-docs/`:
- `plan/2026-11-09-decision-log.md` (Round 14), `plan/2026-11-08-project-plan.md` (ADR backlog 032–034)
- `sessions/007-catalogue-api-observability.md` (brief), `sessions/007a-catalogue-rest-admin-handoff.md`, `sessions/007-catalogue-api-observability-handoff.md` (this file)
- `decisions/ADR-032-problem-details-and-etag-if-match.md`, `decisions/ADR-033-localhost-only-admin-api.md`, `decisions/ADR-034-grpc-server-and-opentelemetry-wiring.md` (new), `decisions/README.md` (index)

`catalogue-service/` (8 commits after `2118d7c`, tag `v0.1.0`, pushed):
- Root: `.gitattributes` (new), `README.md`, `settings.gradle.kts`, `gradle/libs.versions.toml`, `docs/openapi.yaml` (new)
- `application/`: `port/in/PublishEvent.java`, `port/in/GetVenue.java` (new), `ConcurrentUpdateException.java`, `service/EventService.java`, `service/VenueService.java`, `test/.../service/EventServiceTest.java`
- `adapter-rest/` (new module): `build.gradle.kts`.
  - Main: `AdminController`, `EventController`, `ProblemDetailsAdvice`, `Problems`, `ETags`, `LocalhostOnlyAdminInterceptor`, `CorsConfiguration`, `OpenApiConfiguration`, `SeatMapDto`, `EventDto`, `EventSummaryDto`, `PageDto`, `VenueDto`, `CreateVenueRequest`, `CreateEventRequest`.
  - Tests: `RestTestApplication`, `Fixtures`, `EventControllerTest`, `AdminControllerTest`, `LocalhostOnlyAdminInterceptorTest`.
- `adapter-grpc/` (new module): `build.gradle.kts`.
  - Main: `CatalogueGrpcService`, `GrpcErrors`, `ProtoMapper`, `PageTokens`.
  - Tests: `CatalogueGrpcServiceTest`, `DeadlineHandlingTest`, `GrpcErrorsTest`.
- `bootstrap/`: `build.gradle.kts`.
  - Main: `UseCaseConfiguration.java` (observed `@Primary` port beans), `ObservedUseCases.java` (new), `OpenTelemetryLogging.java` (new), `resources/application.properties`, `resources/logback-spring.xml` (new).
  - Tests: `IntegrationTestBase` (new), `RestApiFlowTest` (new), `OpenApiSpecTest` (new), `GrpcApiTest` (new), `TracingTest` (new), `CatalogueApplicationTest` (now extends the base; gRPC port 0, OTLP off).

Outside the project:
- Rows created by the gate check in the platform's `catalogue` database (venue "Gate Hall", event "Gate Check Night") and telemetry in the `otel-lgtm` volume.
- Gradle cache: Spring gRPC, grpc-java 1.85.1, OpenTelemetry, datasource-micrometer, springdoc.
- A throw-away Gradle probe in the session scratchpad (outside the project, used to list starter dependencies).

## Known issues / blockers
1. **Version pins to watch on every Boot upgrade:** `grpc-bom` (must match the contracts' grpc-java), `datasource-micrometer`, and the alpha Logback appender (must match Boot's OTel version). All are in `gradle/libs.versions.toml` with comments.
2. Swagger UI and `/v3/api-docs` are public. Disable or gate them in Phase 2.
3. 500 errors use Spring Boot's default error JSON, not Problem Details (deliberate for now; ADR-032).
4. Logs reach Loki only after Spring has started (the appender is installed then); start-up failures are console-only.
5. gRPC `ListEvents` summaries have no `venue_id` (the read model lacks it); callers use `GetEvent`. `Schedule.ends_at` is never set (no end time in the domain).
6. `bootstrap/build/libs` keeps old `-SNAPSHOT` jars until `clean` (harmless).
7. Carried over: domain events still dropped until the outbox (Phase 2); mavenLocal is the only repository; Javadoc warnings in conventions `arch-rules`.

## What I learned (for interview prep)
REST, Problem Details, ETag/If-Match (412 vs 428), CORS/preflight and OpenAPI are in handoff 007a. 007b:
- **gRPC status codes.** A small fixed set, not HTTP's. Choose by what the client should do:
  - `INVALID_ARGUMENT`: fix the request.
  - `NOT_FOUND`
  - `FAILED_PRECONDITION`: the system state forbids it; don't retry until it changes.
  - `ABORTED`: concurrency conflict; retry the whole read-modify-write.
  - `UNAVAILABLE`: transient; retry with backoff.
  - `DEADLINE_EXCEEDED`
  - `INTERNAL`: a server bug; never leak details.
- **Deadlines.** The client sets an absolute deadline per call, and it travels with the call (`grpc-timeout` header), shrinking as it is passed downstream. When it passes, the client gets `DEADLINE_EXCEEDED` and the server's `Context` is cancelled. A good server checks it so it doesn't burn database time on answers nobody will read. Without deadlines, a slow dependency makes callers pile up.
- **In-process gRPC testing.** grpc-java's in-process transport connects a real client stub to a real server in the same JVM, with no sockets. Serialisation, interceptors, status codes and deadlines all behave as on the network, but tests are fast and need no ports.
- **Traces, spans, context propagation.** A trace is one request's journey; spans are timed steps with a parent, forming a tree. Inside a process the current span lives in a thread-bound context. Between processes it travels in headers (W3C `traceparent` for HTTP and gRPC metadata), so Booking → Catalogue calls will join one trace in Phase 2.
- **Micrometer Observation vs the OpenTelemetry API.** Observation is Spring's instrumentation API: you start or stop an observation once, and handlers turn it into a span (through the tracing bridge to OTel) *and* a timer. The OTel API/SDK is the vendor-neutral model and exporter underneath. We instrument with Observation (what Spring itself uses) and export with the OTel SDK over OTLP.
- **Log/trace correlation.** The active trace id and span id go into the logging MDC (mapped diagnostic context, per-thread fields every log line can print). The console pattern prints them and the OTel appender sends them as fields. In Grafana you jump from a span to its logs and back.
- **RED metrics.** Rate (requests per second), Errors (failed share), Duration (latency distribution, e.g. p95), per endpoint. Every request-driven service needs them. `http_server_requests` and our `catalogue.use-case` timer give all three from one histogram, with `status`/`error` tags.

## Suggested next step
Session 008, `web-shell`:
- React host with Module Federation, listing published events from `GET /events` (CORS allows `http://localhost:5173`).
- Types generated from `catalogue-service/docs/openapi.yaml`.
- Browser OpenTelemetry (or at least `traceparent` on fetches), so the second half of the Phase 1 gate shows **browser → Catalogue → Postgres** as one trace in Tempo.

Needs `make up PROFILES="core"` and `catalogue-service` v0.1.0 running.
