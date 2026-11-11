# Session 008 — `web-shell` + `web-discover` (Module Federation) and Phase 1 gate
Phase: 1 · Estimated: 6–7 hours (split into 008a = both apps working together, 008b = tests, tracing, gate, retro; handoff at the split)

## Goal
A React host (`web-shell`) that loads a remote (`web-discover`) at runtime with Module Federation 2.0, shows published events from `catalogue-service`, and proves the Phase 1 gate: one trace **browser → catalogue-service → Postgres** in Grafana Tempo. Then close Phase 1 with a retro.

## Context to read
- `AGENTS.md`
- `ticketing-docs/plan/2026-11-08-project-plan.md` (frontend rows, Phase 1 gate), `ticketing-docs/plan/2026-11-09-decision-log.md`
- `ticketing-docs/sessions/007-catalogue-api-observability-handoff.md` and `007a-…` (endpoints, CORS, OpenAPI, Tempo/Loki queries)
- `catalogue-service/README.md`, `catalogue-service/docs/openapi.yaml`
- `ticketing-platform/README.md` (OTLP ports)

## Decisions already made (record them; do not re-decide)
- Bundler: **Rsbuild (Rspack)** with **Module Federation 2.0** (`@module-federation/rsbuild-plugin` / `@module-federation/enhanced`).
- Styling: **Tailwind CSS** (each app builds its own CSS; no global style leakage between remotes — explain how).
- TypeScript (strict), **pnpm**, React (current stable), React Router, TanStack Query.
- API types generated from `catalogue-service/docs/openapi.yaml` with `openapi-typescript` (+ `openapi-fetch` if useful).
- Tests: Vitest + Testing Library, **80% coverage gate**; one Playwright e2e.
- Phase 1 shortcut still applies: browser calls Catalogue directly (CORS `http://localhost:5173` already allowed; adjust ports or the allowed origin if the shell runs elsewhere and record it).
- Both repos public on GitHub; **Azad runs `gh repo create`**.

## Part A — housekeeping
1. Append **Round 15** to the decision log (decisions above). Commit with this brief.

## Part B — two apps working together (008a)
2. Two new repos side by side: `web-shell/` and `web-discover/`. Each: `git init` on `master`, `.editorconfig`, `.gitattributes`, `.gitignore`, README, `package.json` with `packageManager` pinned, Node version pinned (`.nvmrc`), ESLint + Prettier (or Biome — pick one, justify) wired into a `check` script.
3. `web-discover` (remote, own dev port):
   - Exposes `./EventList` and `./EventDetail` (or a route module) via Module Federation.
   - `EventList`: published events, paged, soonest first; date shown in the **venue's time zone** (`zoneId`), not the browser's.
   - `EventDetail`: event info + a simple read-only seat-map view (sections → rows → seats) from the JSONB structure.
   - Data via TanStack Query + generated OpenAPI types; Problem Details errors shown nicely (404 page, generic error).
   - Runs standalone too (its own dev entry) for isolated development.
4. `web-shell` (host, port 5173):
   - Layout (header, nav), routes `/` and `/events/:id` that load the remote components at runtime; remote URL from config/env (not hard-coded).
   - React, React DOM, React Router and TanStack Query **shared as singletons** (explain `singleton`, `requiredVersion`, eager vs lazy).
   - Graceful fallback when the remote is down (error boundary + message), proven by a test or manual check.
5. Tailwind in both apps; a tiny shared design convention (colours, spacing) documented in each README — no shared UI library yet.

## Part C — tests, tracing, gate, retro (008b)
6. Vitest + Testing Library in both repos with **80% coverage gate** in `check`; mock API with MSW (Mock Service Worker) using the OpenAPI types.
7. Playwright e2e in `web-shell`: starts shell + remote (and expects catalogue-service running), loads `/`, sees an event, opens its detail. Document how to run it.
8. **Browser tracing:**
   - OpenTelemetry web SDK in the shell (shared to the remote, or initialised once in the host): fetch instrumentation adds W3C `traceparent` to calls to catalogue-service; document-load span optional.
   - Export spans over OTLP/HTTP. If the `otel-lgtm` collector rejects browser requests (CORS), route via the Rsbuild dev-server proxy (e.g. `/otlp` → `http://localhost:4318`) and record why.
   - Check catalogue-service accepts `traceparent` from the browser (CORS must allow that request header; change catalogue-service only if needed, as a small patch release `v0.1.1`).
9. **Gate check part 2:** with `make up PROFILES="core"` + catalogue-service + both apps: open the shell in a browser, load the list and a detail page. In Tempo show **one trace** with the browser fetch span → catalogue HTTP span → use case → SQL. Provide the query/link; **Azad confirms by eye**.
10. ADRs in `ticketing-docs/decisions/` (update index; add numbers to the plan backlog):
    - **ADR-019** Module Federation split (shell + Discover, Checkout, Account; what's shared, runtime remote URLs, failure isolation).
    - **ADR-035** front-end tooling: Rsbuild/Rspack, Tailwind, TanStack Query, generated OpenAPI types, Vitest/MSW/Playwright.
    - **ADR-036** browser tracing (W3C trace context end to end; OTLP export path).
11. **Phase 1 retro** → `ticketing-docs/plan/2026-11-12-phase-1-retro.md`: what was built (repos, tags), gate evidence, hours vs plan (ask Azad for his actual hours), what went well, what to change for Phase 2, open known issues carried forward (collected from handoffs 001–008). Tag `web-shell` and `web-discover` `v0.1.0`.
12. Conventional Commits; push all repos; give Azad the two `gh repo create` commands.

## Acceptance criteria
- Shell loads Discover at runtime; list and detail pages work against catalogue-service; remote-down fallback works.
- `pnpm check` (lint, typecheck, tests with 80% coverage) passes in both repos; Playwright e2e passes.
- One browser → Catalogue → Postgres trace visible in Tempo; Azad confirms. **Phase 1 gate passed.**
- ADR-019, 035, 036 written and indexed; Round 15 recorded; Phase 1 retro written.
- Both web repos tagged `v0.1.0` and pushed; all repos clean.

## Out of scope
- Checkout and Account remotes, Gateway/BFF, Keycloak login (Phase 2).
- Admin UI (use curl / Swagger for creating events).
- Pact consumer tests (Phase 3), mobile.

## Concepts to explain (2–4 lines each)
Module Federation host/remote/exposes/shared, runtime vs build-time integration · singletons and version negotiation for React · why microfrontends exist and their real cost · Rspack vs webpack vs Vite · Tailwind's utility-first approach and how CSS is generated per app · TanStack Query caching and stale-while-revalidate · generating types from OpenAPI · MSW · Playwright vs unit tests · W3C trace context in the browser and why CORS must allow `traceparent` · time zones: showing a venue's local time in a browser in another zone.
