# Handoff 008 — `web-shell` + `web-discover` (Module Federation) and the Phase 1 gate
Date: 2026-11-12

Done in one sitting, with the interim handoff [`008a-web-shell-discover-handoff.md`](008a-web-shell-discover-handoff.md) at the split. 008a has the detail of the two apps; this file covers everything, with 008b in detail. **Phase 1 gate passed** (Azad confirmed the trace by eye).

## Done
**Part A — housekeeping:** decision log **Round 15**, committed with the brief.

**Part B — two apps (008a, summary):**
- `web-discover`, remote, port 5174:
  - Exposes `EventList` and `EventDetail({ id })`.
  - Paged list, soonest first. Dates in the venue's zone. Read-only seat map.
  - Generated OpenAPI types with `openapi-fetch`, TanStack Query, Problem Details errors (404 page, generic error with retry for network errors and 5xx only).
  - Runs standalone too.
- `web-shell`, host, port 5173:
  - Layout and routes. The remote is registered at runtime from `PUBLIC_DISCOVER_URL`.
  - React, React DOM, React Router and TanStack Query shared as singletons (verified: one copy each, `from web_shell`).
  - An error boundary with retry when the remote is down (verified by hand: Discover stopped, then restarted).
- Tailwind v4 in both. Discover's CSS has the prefix `dc:` and no preflight. The design convention is in both READMEs.
- Biome + `tsc` in `pnpm check`.

**Part C — tests, tracing, gate, retro (008b):**
- **Unit tests (Vitest + Testing Library, jsdom), 80% gate in `pnpm check`:**
  - `web-discover`: **29 tests**, coverage lines 94.1 / branches 97.2 / functions 100.
    - **MSW** at the network level. Handlers and fixtures are typed with the generated OpenAPI types, and an unhandled request fails the test.
    - Covers list, paging via `?page=` (with the cached page not refetched), empty list, 5xx with retry, no retry on 4xx, network error, detail with seat map, cancelled event, 404 Problem Details, malformed id (no request), the standalone router, and the federation entries.
  - `web-shell`: **14 tests**, coverage lines 85.4 / branches 83.3 / functions 100.
    - The MF runtime and the remote loader are mocked: registry, `RemoteSlot` (load, remote down + retry, render error), routes, layout survives remote failure, not-found, telemetry.
    - No MSW here: the shell makes no API calls itself.
  - The gate works: it failed the build at 68% branches until I moved env parsing out of `bootstrap.tsx` into a tested function.
- **Playwright e2e** (`web-shell/e2e/browse-events.spec.ts`, Chromium):
  - Starts or reuses both dev servers, opens `/`, picks the first event from the real remote and catalogue-service, opens it, and checks the URL, title, seat map and that the detail API was called.
  - Passes (about 2 s). Needs catalogue-service running with at least one published event.
- **Browser tracing** (`web-shell/src/telemetry.ts`, started in `bootstrap.tsx` before anything fetches):
  - OTel web SDK, **2.x line** (`sdk-trace-web` 2.12.1, `instrumentation-fetch` / `exporter-trace-otlp-http` 0.223.1, `api` 1.9.1).
  - Fetch instrumentation adds `traceparent` only for `PUBLIC_TRACE_PROPAGATE_TO` (`http://localhost:8080`), and ignores `/otlp/`.
  - `BatchSpanProcessor` → OTLP/HTTP JSON to same-origin `/otlp/v1/traces`. The Rsbuild dev server proxies `/otlp` to `http://localhost:4318` (`OTLP_HTTP_ENDPOINT`).
  - Resource: `service.name=web-shell`, `service.version` from `package.json`, `deployment.environment.name=local`.
  - A unit test proves `traceparent` has the W3C format for the API origin, is absent for other origins, and that a span is recorded with that trace id.
- **catalogue-service needed no change** (no `v0.1.1`):
  - Its CORS for `/events/**` already allows any request header.
  - Preflight checked: `Access-Control-Allow-Headers: traceparent`.
  - In the browser, an `OPTIONS` (200) is followed by the `GET` (200).
- **Gate check part 2** (`make up PROFILES="core"` + catalogue-service + both apps). **Confirmed by Azad.** Trace `ea95a7206dcd44d009c2387437b2aa9c`:
  - `web-shell` `GET …/events/{id}` (24 ms)
  - → `catalogue-service` `http get /events/{id}` (13 ms)
  - → `GetEvent` (10 ms)
  - → `connection` (9 ms) → `query` (2 ms)
  - The list call gives the same shape (`http get /events` → `ListEvents` → SQL).
  - TraceQL to find more: `{ resource.service.name = "web-shell" } >> { resource.service.name = "catalogue-service" }`.
- **ADRs:**
  - **ADR-019** (Module Federation split), **ADR-035** (front-end tooling), **ADR-036** (browser tracing).
  - Indexed in `decisions/README.md`; 035 and 036 added to the plan's ADR backlog (019 was already there).
- **Phase 1 retro:** `plan/2026-11-12-phase-1-retro.md`. Repos and tags, gate evidence, hours (Azad: **15–20 h** vs 84 h planned and 33–41 h estimated), what went well, what to change, and 21 known issues carried forward.
- **Releases:** `web-discover` and `web-shell` `v0.1.0` (annotated tags, `package.json` version `0.1.0`, release steps in the READMEs). **Not pushed yet**: the GitHub repos don't exist. Azad runs the commands below.

## Azad: create the two GitHub repos, then push
Auto mode blocks `gh repo create` for me (handoff 003). Run:
```bash
cd ~/learn/java_workspace/distributed-ticketing/web-shell && gh repo create azneotec/web-shell --public --source=. --remote=origin --description "Ticketing web shell: Module Federation host" --push && git push origin --tags
```
```bash
cd ~/learn/java_workspace/distributed-ticketing/web-discover && gh repo create azneotec/web-discover --public --source=. --remote=origin --description "Ticketing Discover micro-frontend (Module Federation remote)" --push && git push origin --tags
```

## Decisions made (and ADRs written)
- ADR-019, ADR-035, ADR-036 (above). Smaller choices (008a choices are in the 008a handoff):
  - **OTel JS 2.x, not 3.x.**
    - 3.0 (2026-10-20) ships `@opentelemetry/sdk-trace` with no web provider, and `sdk-trace-web` stops at 2.12.
    - Mixing lines breaks the `@opentelemetry/api` peer ranges (2.x wants `<1.10`, 0.301 instrumentation wants `^1.10`).
    - The 2.x line is still maintained (`latest-2` dist-tag).
  - **Same-origin `/otlp` proxy, even though the collector allows CORS.**
    - `otel-lgtm` happens to echo any origin. A real collector shouldn't be open to browsers.
    - The browser doesn't need the collector's address, and same-origin exports need no preflight.
  - **`traceparent` only to listed API origins.** Sending it everywhere leaks trace ids and breaks third-party CORS. Our own MF manifest requests carry none.
  - **Tracing set up once in the host**, not per remote: one tracer provider and one fetch patch per page.
  - **No MSW in the shell**, which makes no API calls. Mocking the MF runtime and loader is the right seam there.
  - **Releases:** `package.json` version set to the release in a `chore: release` commit, annotated `vX.Y.Z` tag, next `-SNAPSHOT` with the next change (same shape as the Gradle repos).
  - **Biome 2.6 formats Markdown too.** README changes must pass `pnpm check`. This caught the release commit once; I fixed it before tagging.

## Files created or modified
`ticketing-docs/`:
- `plan/2026-11-09-decision-log.md` (Round 15), `plan/2026-11-08-project-plan.md` (ADR backlog 035–036), `plan/2026-11-12-phase-1-retro.md` (new)
- `decisions/ADR-019-module-federation-split.md`, `decisions/ADR-035-frontend-tooling.md`, `decisions/ADR-036-browser-tracing.md` (new), `decisions/README.md` (index)
- `sessions/008-web-shell-discover.md` (brief, committed), `sessions/008a-web-shell-discover-handoff.md`, `sessions/008-web-shell-discover-handoff.md` (this file)

`web-discover/` (new repo, 4 commits, tag `v0.1.0`, not pushed):
- Root: `.editorconfig`, `.env`, `.gitattributes`, `.gitignore`, `.nvmrc`, `README.md`, `biome.json`, `package.json`, `pnpm-lock.yaml`, `postcss.config.mjs`, `rsbuild.config.ts`, `tsconfig.json`, `vitest.config.ts`
- `src/`: `index.ts`, `bootstrap.tsx`, `StandaloneApp.tsx`, `StandaloneApp.test.tsx`, `env.d.ts`, `format.ts`, `format.test.ts`, `styles.css`, `standalone.css`
- `src/api/`: `schema.d.ts` (generated), `client.ts`, `client.test.ts`, `queries.ts`
- `src/components/`: `EventList.tsx`, `EventList.test.tsx`, `EventDetail.tsx`, `EventDetail.test.tsx`, `SeatMapView.tsx`, `ErrorView.tsx`, `Status.tsx`
- `src/exposed/`: `EventList.tsx`, `EventDetail.tsx`, `exposed.test.ts`
- `src/test/`: `setup.ts`, `server.ts`, `fixtures.ts`, `render.tsx`

`web-shell/` (new repo, 5 commits, tag `v0.1.0`, not pushed):
- Root: the same set as above, plus `playwright.config.ts`
- `e2e/browse-events.spec.ts`
- `src/`: `index.ts`, `bootstrap.tsx`, `App.tsx`, `App.test.tsx`, `env.d.ts`, `styles.css`, `telemetry.ts`, `telemetry.test.ts`, `layout/Layout.tsx`, `pages/pages.tsx`, `test/setup.ts`
- `src/remotes/`: `registry.ts`, `registry.test.ts`, `RemoteSlot.tsx`, `RemoteSlot.test.tsx`, `RemoteErrorBoundary.tsx`

Outside the repos:
- `catalogue` database: venues "Budokan Hall" (Tokyo), "Harbor Arena" (New York); published events "Test Concert 1–11" (created via the localhost admin API with a throw-away scratchpad script).
- Telemetry in the `otel-lgtm` volume, including the gate trace.
- Playwright Chromium in `~/Library/Caches/ms-playwright` (~100 MB headless shell + Chromium).
- pnpm store: front-end packages.

## Known issues / blockers
1. **Both web repos are not on GitHub yet.** Azad runs the two commands above. `ticketing-docs` is pushed.
2. Shell ↔ remote contract (module names, props) isn't type-checked across repos (MF `dts` off); the e2e test covers it.
3. `PUBLIC_*` URLs are baked in at build time. Runtime config is needed for Kubernetes (Phase 4).
4. `/otlp` exists only on the dev server. A production build needs the gateway route (Phase 2).
5. In dev, React StrictMode double-mounts, so Tempo shows some browser-only `GET` spans (requests cancelled before reaching the server). Noise only.
6. Standalone Discover (`:5174`) needs `CATALOGUE_CORS_ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174`.
7. TypeScript 5.9 (until `openapi-typescript` supports 7) and the OTel JS 2.x line (until 3.x has a web SDK).
8. `bootstrap.tsx` in both apps isn't unit-tested (wiring only; the e2e test covers it).
9. Playwright `webServer` reuses running dev servers; started from scratch it needs `pnpm install` in `web-discover` too.
10. Carried over from earlier handoffs: the full list is in the Phase 1 retro (mavenLocal only, outbox, Swagger public, 500s not Problem Details, and so on).

## What I learned (for interview prep)
Module Federation, singletons, micro-frontend costs, Rspack/webpack/Vite, Tailwind, TanStack Query, OpenAPI types and time zones are in handoff 008a. 008b:
- **MSW (Mock Service Worker).**
  - It intercepts requests at the network layer (a Service Worker in the browser, interceptors on `fetch`/`http` in Node) and answers from handlers.
  - The app's real HTTP client runs unchanged: URL building, headers, JSON parsing and error handling are all tested, not mocked away.
  - With typed handlers, the fake API follows the real contract.
- **Playwright vs unit tests.**
  - Unit/component tests (Vitest, jsdom) are fast and precise and run on every `check`, but they fake the browser and the network.
  - Playwright drives a real browser against real servers: slow and needs the stack, but it is the only check that the shell, the remote, CORS and the service work together.
  - The usual shape is many unit tests and very few e2e tests: one per critical user journey.
- **W3C trace context in the browser.**
  - `traceparent: 00-<32-hex trace id>-<16-hex parent span id>-<flags>` tells the server "you are a child of this span in this trace". `01` means sampled.
  - The browser creates the root span, so the trace starts where the user acts and shows network time (browser span minus server span).
- **Why CORS must allow `traceparent`.**
  - A custom request header makes a cross-origin request "non-simple", so the browser first sends an `OPTIONS` preflight asking whether the header is allowed.
  - If the server doesn't list it in `Access-Control-Allow-Headers`, the browser never sends the real request: tracing would break the app.
  - That's also why we only add the header for our own APIs.
- **Exporting browser telemetry.**
  - Browsers can't speak gRPC, so OTLP/HTTP (JSON) is used.
  - Spans are batched to save requests. A same-origin route keeps the collector private and avoids a preflight.
  - Browser telemetry is untrusted input: anyone can send fake spans.
- **Coverage gates bite honestly.** 80% branch coverage failed on env fallbacks inside untestable start-up code. The fix was to move logic out of the entry file into a small tested function, not to lower the bar or exclude the file.

## Suggested next step
- Azad: run the two `gh repo create` commands (above).
- Elephant: review the Phase 1 retro, especially "What to change for Phase 2" (deliberate breaks, Azad hands-on steps, CI before more repos, re-plan by sessions), and write the Phase 2 kickoff brief.
- A small CI session (GitHub Actions running `./gradlew build` and `pnpm check` per repo, GitHub Packages instead of mavenLocal) would pay off before Inventory/Booking/Payment triple the repo count.
