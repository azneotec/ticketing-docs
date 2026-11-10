# Handoff 007a — `catalogue-service` REST + admin API
Date: 2026-11-11

Interim handoff at the split the brief asks for (Parts A and B). 007b (gRPC, observability, gate check, ADRs, release) follows; its handoff will be `007-catalogue-api-observability-handoff.md` and will repeat anything still relevant from here.

## Done
**Part A — housekeeping**
- Decision log **Round 14** recorded; committed with brief 007 and pushed (`ticketing-docs` `5f88705`).
- `catalogue-service/.gitattributes` copied from the other repos (LF everywhere, CRLF for `*.bat`).

**Part B — REST + admin (`catalogue-service`, pushed to `master`)**
- **Port change (application):**
  - `PublishEvent.publishEvent(EventId id, long expectedVersion)`: the service compares the stored version with the caller's and throws `ConcurrentUpdateException` before changing anything. Without it, `If-Match` could not be checked against the stored version.
  - New `GetVenue` use case (in `VenueService`), so the `Location` of a new venue resolves.
  - Application coverage still 100%.
- **New module `adapter-rest`** (Spring MVC, `spring-boot-conventions` for the BOM only, no boot jar), package `…catalogue.adapter.rest`:

  | Method + path | Result |
  |---|---|
  | `GET /events?status=PUBLISHED,CANCELLED&page=0&size=20` | Paged summaries, soonest first. Default `PUBLISHED`. `DRAFT` → 400. `size` 1–100. |
  | `GET /events/{id}` | Detail with seat map, `ETag: "<version>"`. Drafts → 404. `If-None-Match` → 304. |
  | `POST /admin/venues` | 201 + `Location` |
  | `GET /admin/venues/{id}` | 200 / 404 |
  | `POST /admin/events` | 201 + `Location` + `ETag: "0"` (draft) |
  | `GET /admin/events/{id}` | Any status, with `ETag` (so an organiser can re-read the version) |
  | `POST /admin/events/{id}/publish` | Needs `If-Match`; 200 + new `ETag` |

  - Request/response DTOs are records local to the adapter (`SeatMapDto`, `EventDto`, `EventSummaryDto`, `PageDto`, `VenueDto`, `Create*Request`). No domain types are serialised. `EventDto` flattens the sealed status with record patterns (`case Published(var publishedAt) -> …`).
  - Bean Validation checks shape only (present, non-empty, nested `@Valid`); the domain keeps the rules (lengths, uniqueness) and its `InvalidValueException` becomes 400. One place per rule.
- **Problem Details** (`ProblemDetailsAdvice extends ResponseEntityExceptionHandler`), `type` URNs `urn:ticketing:problem:<slug>`:
  - Exhaustive switch over sealed `DomainException`: `InvalidValueException` 400, `IllegalTransitionException` 409.
  - `NotFoundException` 404, `ConcurrentUpdateException` **412**.
  - Missing or `*` `If-Match` **428**; malformed (`3`, `W/"3"`, lists) 400.
  - Validation errors list every field (`errors: [{field, message}]`), for bodies and query parameters.
  - Spring MVC's own errors (unreadable JSON, bad UUID, unknown enum) are Problem Details via the base class.
- **Localhost-only admin:** `LocalhostOnlyAdminInterceptor` on `/admin`, `/admin/**`. Remote address must be loopback (`InetAddress.ofLiteral(..).isLoopbackAddress()`, no DNS); otherwise 403 Problem Details. `X-Forwarded-For` is ignored (tested).
- **CORS:** `/events/**` only, `GET`/`HEAD`, exposes `ETag`, origins from `catalogue.cors.allowed-origins` (env `CATALOGUE_CORS_ALLOWED_ORIGINS`, default `http://localhost:5173`). No CORS mapping for `/admin`.
- **OpenAPI:** springdoc **3.2.0** (built against Boot 4.1.2). Swagger UI at `/swagger-ui.html`, spec at `/v3/api-docs(.yaml)`. `docs/openapi.yaml` committed; `OpenApiSpecTest` fails when it differs from the running app. Regenerate: `./gradlew :bootstrap:test --tests '*OpenApiSpecTest' -PupdateOpenApi`. Responses annotated (201/304/4xx with `application/problem+json`) so the spec is accurate for the web shell.
- **Tests:**
  - `adapter-rest`: 48 `@WebMvcTest` slice tests with mocked ports (`MockMvcTester`, `@MockitoBean`), coverage 96.1%.
  - `bootstrap`: `RestApiFlowTest` over real HTTP + Testcontainers Postgres: venue → draft (ETag `"0"`) → hidden from buyers → 428 → publish with `If-Match: "0"` → listed, ETag `"1"` → stale `"0"` gives 412 → `"1"` gives 409 (already published).
  - `IntegrationTestBase` shares one Postgres container and one Spring context across bootstrap tests.
  - `./gradlew build` green: 105 tests, coverage domain 100, application 100, persistence 98.5, rest 96.1, bootstrap 100.

## Decisions made
- **412 for a stale `If-Match`, 409 for illegal transitions.** RFC 9110 defines 412 for a failed conditional request. Since every state-changing admin call must carry `If-Match`, any version conflict is a failed precondition. Keeping 409 for business-rule conflicts lets a client tell "reload and retry" from "this will never work". (ADR-032 in 007b.)
- **`If-Match: *` is treated as missing (428).** `*` matches any existing version, which would switch off lost-update protection.
- **Admin guard is a Spring MVC interceptor, not a servlet filter.** The decision ("loopback only, 403 otherwise") is unchanged; the interceptor is matched by the same path patterns as the controllers, so no path spelling (`//admin`, `/admin;x`) reaches an admin controller without it. (ADR-033 in 007b.)
- **Public `GET /events/{id}` hides drafts (404)**, and the public list refuses `DRAFT`. Drafts are only for organisers (domain doc on `Draft`). Organisers read them at `GET /admin/events/{id}`.
- **gRPC server (looked up early for 007b):** Spring Boot 4.1 ships gRPC itself (`spring-boot-starter-grpc-server`, Spring gRPC 1.1.2 in the Boot BOM), so 007b uses it, not plain grpc-java.

## Files created or modified
`ticketing-docs/`:
- `plan/2026-11-09-decision-log.md` (Round 14), `sessions/007-catalogue-api-observability.md` (brief, committed), `sessions/007a-catalogue-rest-admin-handoff.md` (this file)

`catalogue-service/`:
- `.gitattributes` (new), `settings.gradle.kts`, `gradle/libs.versions.toml`, `bootstrap/build.gradle.kts`, `bootstrap/src/main/resources/application.properties`
- `application/src/main/java/.../application/port/in/PublishEvent.java`, `.../port/in/GetVenue.java` (new), `.../ConcurrentUpdateException.java`, `.../service/EventService.java`, `.../service/VenueService.java`
- `application/src/test/java/.../service/EventServiceTest.java`
- `adapter-rest/build.gradle.kts` (new)
- `adapter-rest/src/main/java/io/github/azneotec/ticketing/catalogue/adapter/rest/` (all new): `EventController.java`, `AdminController.java`, `ProblemDetailsAdvice.java`, `Problems.java`, `ETags.java`, `LocalhostOnlyAdminInterceptor.java`, `CorsConfiguration.java`, `OpenApiConfiguration.java`, `SeatMapDto.java`, `EventDto.java`, `EventSummaryDto.java`, `PageDto.java`, `VenueDto.java`, `CreateVenueRequest.java`, `CreateEventRequest.java`
- `adapter-rest/src/test/java/.../adapter/rest/` (all new): `RestTestApplication.java`, `Fixtures.java`, `EventControllerTest.java`, `AdminControllerTest.java`, `LocalhostOnlyAdminInterceptorTest.java`
- `bootstrap/src/test/java/io/github/azneotec/ticketing/catalogue/`: `IntegrationTestBase.java`, `RestApiFlowTest.java`, `OpenApiSpecTest.java` (new); `CatalogueApplicationTest.java` (now extends the base)
- `docs/openapi.yaml` (new, generated)

## Known issues / blockers
1. Swagger UI and `/v3/api-docs` are public (not localhost-only). Fine locally; Phase 2 should disable them outside dev or put them behind the gateway.
2. The server binds all interfaces (`0.0.0.0:8080`), so the localhost-only guard is what protects `/admin`, not the bind address. Intended (the guard keeps working inside containers), recorded for ADR-033.
3. Unexpected exceptions (500) still use Spring Boot's default `/error` JSON, not Problem Details. Deliberately not caught: a catch-all could leak internals. Suggestion for later.
4. Carried over: domain events still dropped until the outbox (Phase 2).

## What I learned (for interview prep)
- **REST resource design and status codes.** Nouns as resources (`/events/{id}`), verbs only for state changes that aren't plain field updates (`/publish`). 201 + `Location` for creation, 404 when it's not there (or not visible to you), 409 when the state forbids it, 412/428 for preconditions.
- **Problem Details (RFC 9457).** One JSON error shape (`type`, `title`, `status`, `detail`, `instance`, plus extensions such as `errors`) with media type `application/problem+json`. Clients switch on the stable `type` URI, not on message text.
- **ETag / If-Match.** The ETag is a version token for a representation. `If-None-Match` on reads saves bandwidth (304). `If-Match` on writes makes them conditional: the server applies the change only if the resource is still at that version, so two organisers can't silently overwrite each other (optimistic locking over HTTP). **412** = you sent a precondition and it failed; **428** = this server requires a precondition and you sent none.
- **CORS and preflight.** Browsers block a page from reading responses from another origin unless the server allows it with `Access-Control-Allow-Origin`. For non-simple requests the browser first sends an `OPTIONS` *preflight*; no allow headers in the answer means the real request is never sent. CORS protects browser users, not the server: `curl` ignores it, which is why the admin API also needs the loopback guard.
- **OpenAPI and why commit the spec.** A machine-readable description of the API. Generating it from the code keeps it true; committing it makes API changes visible in diffs and lets the web shell generate types without running the service. The test turns "forgot to update the spec" into a failing build.
