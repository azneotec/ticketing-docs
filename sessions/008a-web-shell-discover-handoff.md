# Handoff 008a — `web-shell` + `web-discover` working together
Date: 2026-11-12

Interim handoff at the split the brief asks for (Parts A and B). 008b (tests, browser tracing, Phase 1 gate, ADRs, retro, tags) follows; its handoff will be `008-web-shell-discover-handoff.md` and will repeat anything still relevant from here.

## Done
**Part A — housekeeping**
- Decision log **Round 15** recorded and committed together with brief 008 (`ticketing-docs` `617a892`, not pushed yet).

**Part B — two apps (new local repos, `master`, one commit each, not pushed: Azad creates the GitHub repos in 008b)**
- **Both repos:**
  - `.editorconfig`, `.gitattributes` (LF), `.gitignore`, `.nvmrc` (26), README.
  - `packageManager: pnpm@12.18.0`, `engines.node >=26`.
  - Biome 2.6.6 (lint + format + import sorting) and `tsc --noEmit`, wired into `pnpm check`.
  - TypeScript strict, plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`.
- **`web-discover`** (remote, port **5174**, `web-discover` `c0e5117`):
  - Exposes `./EventList` and `./EventDetail` (props `{ id }`; the host owns the route).
  - `EventList`: published events, soonest first, 10 per page, `?page=` in the URL, `keepPreviousData` while paging.
  - `EventDetail`: title, cancelled notice, date, capacity, read-only seat map (sections → rows → seats).
  - **Dates in the venue's zone:** `Intl.DateTimeFormat({ timeZone: zone })` on `startsAt`. Checked: Tokyo 10:00Z shows "19:00 GMT+9", New York shows "05:00 GMT-5".
  - **API types:** generated from `../catalogue-service/docs/openapi.yaml` with `openapi-typescript` into `src/api/schema.d.ts` (committed; `pnpm generate:api` regenerates). Calls go through `openapi-fetch`.
  - **Problem Details errors:**
    - 404 shows "Event not found". A malformed id is treated as 404 with no request.
    - Other errors show the server's `detail`, with "Try again" only when retrying can help (network or 5xx).
    - TanStack Query retries network errors and 5xx only.
  - **Standalone dev entry** (`src/bootstrap.tsx`, `StandaloneApp`), with its own router and QueryClient.
- **`web-shell`** (host, port **5173**, `web-shell` `93c7d2d`):
  - Layout (header, nav, footer). Routes `/`, `/events/:id`, and a `*` not-found page.
  - **Remotes registered at runtime:** `registerRemotes` reads `PUBLIC_DISCOVER_URL` from `.env` and points at `<url>/mf-manifest.json`. No remotes in the build config.
  - `react`, `react-dom`, `react-router`, `@tanstack/react-query` shared as **singletons**. Verified in the browser: `__FEDERATION__.__SHARE__` shows each loaded once, `from web_shell`, used in both apps.
  - **Async boundary:** `index.ts` only does `import('./bootstrap')`.
  - **Remote-down fallback:** `RemoteErrorBoundary` + `RemoteSlot`. Checked by hand:
    - Stopped Discover and reloaded: header and nav stayed, the slot showed "This section is unavailable".
    - Restarted Discover and pressed **Try again**: the page recovered. A new `React.lazy` is made per attempt, because lazy caches a failed import.
- **Tailwind v4 in both apps:**
  - The shell has full Tailwind, with preflight (the reset).
  - Discover's exposed CSS has the prefix `dc` (`dc:flex`, `--dc-color-*`) and **no preflight**.
  - Checked in the built CSS: no `box-sizing` reset, and variables are prefixed.
  - Design convention (colours, spacing, shape, type) written down in both READMEs.
- **Manual check against catalogue-service:** list, paging (page 2 of 2), detail with seat map, unknown id, malformed id, remote down and recovery, standalone Discover.

## Decisions made
- **Biome over ESLint + Prettier:** one fast tool and one config (like Spotless in the Java repos). Its recommended rules cover the React hooks and a11y checks we need. To record in ADR-035.
  - Trap: `biome migrate` rewrote `"recommended": true` to `"preset": "none"` (all rules off). I set it to `"preset": "recommended"`.
- **TypeScript 5.9.3, not 7.0:** `openapi-typescript` 7.13 declares `typescript ^5.x`, and `msw` needs ≥ 5.9.
- **pnpm 12 minimum release age:** pnpm refuses versions newer than its age threshold (supply-chain protection), unless a version is forced. Forcing Biome 2.6.7 made pnpm write an exclusion list. I removed it and let pnpm choose (2.6.6). Rule for next time: don't pin brand-new versions.
- **Remote port 5174**, and catalogue-service's **CORS default left unchanged**:
  - The shell (5173) is the only browser origin in normal use.
  - Running Discover standalone needs `CATALOGUE_CORS_ALLOWED_ORIGINS=http://localhost:5173,http://localhost:5174` (documented in the Discover README). No code change.
- **`traceparent` already allowed by catalogue-service CORS.** Spring's default CORS registration permits all request headers. Preflight checked: `Access-Control-Allow-Headers: traceparent`. A `v0.1.1` patch is probably **not** needed; 008b confirms with real browser tracing.
- **`EventDetail` takes `id` as a prop** instead of reading the route itself, so the host owns URLs and the remote has no route contract beyond links to `/events/:id`.
- **`openapi-fetch` looks up `fetch` per call** (`fetch: (r) => globalThis.fetch(r)`), so OpenTelemetry and MSW, which wrap `globalThis.fetch` later, still see the calls.
- **Remote URL is a build-time env var** (`PUBLIC_DISCOVER_URL`). Registration is at runtime, so switching to a runtime `config.json` for Kubernetes (Phase 4) changes one function.

## Files created or modified
`ticketing-docs/`:
- `plan/2026-11-09-decision-log.md` (Round 15), `sessions/008-web-shell-discover.md` (brief, committed), `sessions/008a-web-shell-discover-handoff.md` (this file)

`web-discover/` (new repo):
- Root: `.editorconfig`, `.env`, `.gitattributes`, `.gitignore`, `.nvmrc`, `README.md`, `biome.json`, `package.json`, `pnpm-lock.yaml`, `postcss.config.mjs`, `rsbuild.config.ts`, `tsconfig.json`
- `src/`: `index.ts`, `bootstrap.tsx`, `StandaloneApp.tsx`, `env.d.ts`, `format.ts`, `styles.css`, `standalone.css`
- `src/api/`: `schema.d.ts` (generated), `client.ts`, `queries.ts`
- `src/components/`: `EventList.tsx`, `EventDetail.tsx`, `SeatMapView.tsx`, `ErrorView.tsx`, `Status.tsx`
- `src/exposed/`: `EventList.tsx`, `EventDetail.tsx`

`web-shell/` (new repo):
- Root: same set as above
- `src/`: `index.ts`, `bootstrap.tsx`, `App.tsx`, `env.d.ts`, `styles.css`, `layout/Layout.tsx`, `pages/pages.tsx`, `remotes/registry.ts`, `remotes/RemoteSlot.tsx`, `remotes/RemoteErrorBoundary.tsx`

Outside the repos:
- Local test data in the platform's `catalogue` database: venues "Budokan Hall" (Tokyo) and "Harbor Arena" (New York), and published events "Test Concert 1–11" (Jan 2027). Created with a throw-away seed script in the session scratchpad, via the localhost admin API.
- pnpm store: front-end packages.

## Known issues / blockers
1. No tests yet: Vitest, MSW, the 80% gate and Playwright are 008b. `pnpm check` is lint + typecheck only for now.
2. Remote types in the shell are hand-written (`ComponentType<{ id: string }>`); MF's `dts` type sharing is off. Acceptable with two exposed modules; revisit when Checkout arrives.
3. `PUBLIC_*` env vars are baked in at build time; a deployed build needs a rebuild per environment until runtime config (Phase 4).
4. The Discover dev server allows CORS only from `SHELL_ORIGIN` (default `http://localhost:5173`).
5. Nothing pushed yet: Azad runs the two `gh repo create` commands in 008b.

## What I learned (for interview prep)
- **Module Federation (host, remote, exposes, shared).**
  - A *remote* is a separately built and deployed app that *exposes* modules (`./EventList`).
  - A *host* loads them over the network at runtime through the remote's manifest.
  - *Shared* libraries are negotiated so both use one copy.
  - Runtime integration means teams deploy independently. Build-time integration (an npm package) means a rebuild and redeploy of the host for every change.
- **Singletons and version negotiation.**
  - `singleton: true` means one copy only, whatever versions are asked for. That is essential for React (two copies break hooks) and for context-based libraries (router, query client).
  - `requiredVersion` is the range a consumer accepts; a mismatch warns (or fails with `strictVersion`).
  - Shared modules are loaded lazily behind an async boundary (`import('./bootstrap')`); `eager` puts them in the entry chunk and skips negotiation.
- **Why micro-frontends, and their cost.**
  - Gain: independent teams, deploys and failure isolation per feature.
  - Cost: runtime coupling through shared versions, more repos and pipelines, duplicate code, harder end-to-end testing, a slower first load, and cross-app UX consistency to maintain.
  - Worth it only when team or deploy independence is the bottleneck. Here it's a learning goal, recorded honestly in ADR-019.
- **Rspack vs webpack vs Vite.**
  - webpack: the mature JavaScript bundler, where Module Federation was born.
  - Rspack: webpack's architecture and plugin API re-implemented in Rust, many times faster, with Module Federation built in.
  - Vite: native ES modules in dev and Rollup/Rolldown for builds, with a great dev experience, but federation is a community plugin.
  - Rsbuild is Rspack with sensible defaults (like Vite is to Rollup).
- **Tailwind's utility-first approach.**
  - Small single-purpose classes (`p-4`, `text-slate-600`) composed in markup instead of hand-written CSS.
  - At build time Tailwind scans the source for class names and generates only those, so each app ships just what it uses.
  - A prefix plus no preflight keeps a remote's CSS from touching the host.
- **TanStack Query.**
  - A server-state cache keyed by query keys (`['events','page',{page,size}]`).
  - Stale-while-revalidate: show cached data immediately, refetch in the background when stale (`staleTime` 30 s), and update.
  - It also deduplicates requests, retries, cancels via `AbortSignal`, and keeps the previous page while the next one loads.
- **Types from OpenAPI.**
  - The service's committed spec is the contract. `openapi-typescript` turns it into TypeScript types, and `openapi-fetch` checks paths, parameters and responses at compile time.
  - Drift shows up as a type error after `pnpm generate:api`, not as a runtime bug.
- **Time zones in the browser.**
  - Store and send instants (UTC, `startsAt`) plus the venue's IANA zone (`Asia/Tokyo`).
  - Format with `Intl.DateTimeFormat({ timeZone })`, so everyone sees the time printed on the ticket, not their own local time.

## Suggested next step
008b, in this order:
1. Vitest + Testing Library + MSW in both repos with the 80% gate in `check`.
2. Playwright e2e in the shell.
3. OpenTelemetry web SDK in the shell (resolve the `@opentelemetry/api` 1.9/1.10 peer split between `sdk-trace-web` 2.12 and `instrumentation-fetch` 0.301) with the OTLP proxy.
4. Gate check in Tempo (Azad confirms).
5. ADR-019/035/036, the Phase 1 retro (ask Azad's hours), tags `v0.1.0`, `gh repo create`, push.
