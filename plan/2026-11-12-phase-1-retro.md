# Phase 1 retro — Foundations
Date: 2026-11-12 · Sessions 001–008 (2026-11-08 → 2026-11-12)

## What was built
| Repo | What | Release |
|---|---|---|
| `ticketing-docs` | Plan, decision log (rounds 1–15), session briefs and handoffs, ADR template + ADRs 001–007, 010, 016, 019, 027–036 | (docs, untagged) |
| `ticketing-build-conventions` | Gradle convention plugin: Java 25 toolchain, JaCoCo 80%, ArchUnit rules, Spotless, versioning from Git tags | `v0.1.2` |
| `ticketing-contracts` | Protobuf (Buf) + Avro schemas, versioned artifacts, breaking-change checks | `v0.1.1` |
| `ticketing-platform` | Compose: shared Postgres (database per service), Kafka KRaft, `grafana/otel-lgtm`; memory limits; smoke test | (untagged) |
| `catalogue-service` | Hexagonal service: domain, JPA + JSONB, Flyway, REST + Problem Details + ETag, localhost-only admin, gRPC, OpenTelemetry | `v0.1.0` |
| `web-discover` | Module Federation remote: event list and detail with seat map, generated OpenAPI types, TanStack Query, MSW tests | `v0.1.0` |
| `web-shell` | Module Federation host: layout, routing, shared singletons, failure isolation, browser tracing, Playwright e2e | `v0.1.0` |

## Gate evidence
**Gate:** request traced browser → Catalogue → Postgres in Grafana/Tempo. **Passed 2026-11-12, confirmed by Azad by eye.**
- **Part 1 (session 007):** REST admin flow and gRPC traced in Tempo (`http post /admin/events/{id}/publish` → `PublishEvent` → SQL; gRPC `GetEvent` → SQL), log line correlated in Loki, RED metrics in Prometheus.
- **Part 2 (session 008):** trace `ea95a7206dcd44d009c2387437b2aa9c`:
  - `web-shell` `GET http://localhost:8080/events/{id}` (24 ms)
  - → `catalogue-service` `http get /events/{id}` (13 ms)
  - → `GetEvent` (10 ms)
  - → `connection` (9 ms) → `query` (2 ms)
- Find it again: Grafana → Explore → Tempo → TraceQL `{ resource.service.name = "web-shell" } >> { resource.service.name = "catalogue-service" }`.

## Hours vs plan
| | Hours |
|---|---|
| Plan (roadmap: weeks 1–6 × 14 h/week) | 84 |
| Sum of session brief estimates (001–008) | 33–41 |
| **Actual (Azad)** | **15–20** |

Phase 1 took about a fifth to a quarter of the planned time, and about half of the session estimates, over 5 calendar days instead of 6 weeks. The work was done by goldfish sessions; Azad's hours went into planning, reviewing handoffs, approving plans and checking by eye. **Caveat:** fast building is not the same as learning. The plan's rhythm is "concept → minimal build → deliberate break → measure → ADR". Phase 1 built and measured, but did very few deliberate breaks, and the "What I learned" sections were read, not practised.

## What went well
- **Elephant/goldfish loop.** Briefs with "decisions already made" stopped sessions from re-deciding things. Handoffs with known issues gave each new session the real state, and the split handoffs (007a, 008a) were natural checkpoints.
- **Quality gates from day one:** formatting, ArchUnit, 80% coverage, an OpenAPI spec test and type-generated clients. Most mistakes were caught by the build, not by Azad.
- **Contracts first.** The committed OpenAPI spec made the web client a type-checked consumer on the first try. Protobuf contracts did the same for gRPC.
- **Observability early.** Because tracing existed in the service, the browser half of the gate was a small addition (the existing CORS already allowed `traceparent`, so no service patch).
- **Constrained hardware held.** Core platform about 0.9 GB, catalogue about 280 MB RSS. Nothing ran that wasn't needed.

## What to change for Phase 2
1. **Build in the deliberate breaks.** Each Phase 2 brief should include one "break it on purpose" step with a measurement (for example: kill Booking mid-saga, stop Kafka, slow Postgres) and ask Azad to predict the outcome first.
2. **More Azad hands-on.** For each brief, mark one or two steps "Azad types this" (a TraceQL query, an `If-Match` curl, a gRPC call), so the knowledge sticks.
3. **Re-plan the calendar.** Phase 1 was ~5× faster than planned. Keep the phase gates, but plan Phase 2 by sessions, not weeks, and use the saved time on understanding: re-explaining ADRs aloud as interview practice.
4. **Pin fewer brand-new versions.** Several young tool lines changed under us (OTel JS 3.x without a web SDK, MSW 3 option rename, Biome migration that disabled rules, TypeScript 7 vs `openapi-typescript`). Prefer the newest version that is at least a few weeks old (pnpm's minimum release age already does this).
5. **Expect `gh repo create` to be Azad's step** in every brief that creates a repo (auto mode blocks it; handoff 003).
6. **CI is overdue.** Everything is checked on one laptop; mavenLocal is the only artifact repository. Phase 2 should add GitHub Actions + GitHub Packages before the number of repos doubles.

## Open known issues carried into Phase 2
From handoffs 001–008. Issues already resolved are left out.
**Build and supply chain**
1. mavenLocal is the only Maven repository. Every machine must `publishToMavenLocal` conventions and contracts first. CI needs GitHub Packages. (003, 004)
2. Conventions README "Releasing" step 6 should say to `publishToMavenLocal` after bumping to `-SNAPSHOT`. (004)
3. Buf must be installed on every machine (and in CI). (004)
4. Gradle configuration cache not enabled. (003)
5. Javadoc "no @return" warnings in conventions `arch-rules`. (005)
6. Version pins to recheck on every Spring Boot upgrade: `grpc-bom`, `datasource-micrometer`, the alpha OTel Logback appender. (007)
7. Front end on TypeScript 5.9 (until `openapi-typescript` supports TS 7) and the OTel JS 2.x line (until 3.x has a web SDK). (008)

**Service behaviour**
8. Domain events are recorded but dropped until the outbox (Phase 2, ADR-012). (006)
9. 500 errors use Spring's default error JSON, not Problem Details. (007)
10. Swagger UI and `/v3/api-docs` are public. Gate or disable them in Phase 2. (007)
11. Admin API is localhost-only, not authenticated (ADR-033). Replaced by Keycloak roles in Phase 2. (007)
12. gRPC `ListEvents` summaries lack `venue_id`; `Schedule.ends_at` is never set. (007)
13. Logs reach Loki only after Spring has started. (007)

**Front end**
14. The shell ↔ remote contract (module names, props) is not type-checked across repos. Covered by the e2e test only. (008)
15. `PUBLIC_*` URLs are baked in at build time. Runtime config is needed for Kubernetes (Phase 4). (008)
16. `/otlp` exists only on the dev server. A production build needs the gateway route (Phase 2). (008)
17. Standalone Discover (`:5174`) needs `CATALOGUE_CORS_ALLOWED_ORIGINS` to include its origin. (008)

**Environment and housekeeping**
18. Homebrew `node` installed next to nvm's Node; consider removing it. (001)
19. Check that `azneotech@gmail.com` is verified on the `azneotec` GitHub account, so commits link to the profile. (002)
20. `otel-lgtm` runs Pyroscope too; idle memory grows with stored data. (005)
21. Local test data in the `catalogue` database: "Gate Check Night" (007) and "Test Concert 1–11" (008).
