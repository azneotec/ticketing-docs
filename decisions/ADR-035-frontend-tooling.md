# ADR-035: Front-end tooling: Rsbuild, Tailwind, TanStack Query, generated API types, Vitest/MSW/Playwright

- **Status:** Accepted
- **Date:** 2026-11-12 (decided in round 15; built in session 008)

## Context
Phase 1 starts the web front end (`web-shell`, `web-discover`). The tools must support Module Federation well, keep each app's CSS isolated, give type-safe calls to the services' REST APIs, and meet the same quality bar as the Java services: one `check` command, an 80% coverage gate, formatting enforced by the build.

## Decision
- **Build:** Rsbuild (on Rspack) with `@module-federation/rsbuild-plugin` (MF 2.0). Node 26 and pnpm 12, pinned per repo (`.nvmrc`, `packageManager`).
- **Language:** TypeScript strict, plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`. TypeScript **5.9**, because `openapi-typescript` declares `typescript ^5.x`; we move to TypeScript 7 when it is supported.
- **Lint and format:** **Biome** (recommended rules, formatter, import sorting), one config per repo, run by `pnpm check`.
- **UI:** React, React Router, **Tailwind CSS v4**, built per app. Remotes use a prefix and no preflight (ADR-019).
- **Data:** **TanStack Query** for server state, with `staleTime` 30 s, retries for network errors and 5xx only, and the previous page kept while paging.
- **API types:** generated from each service's committed OpenAPI spec with `openapi-typescript`. Calls go through `openapi-fetch`. Errors are read as Problem Details (RFC 9457). `fetch` is looked up per call, so tracing and MSW can wrap it.
- **Tests:**
  - **Vitest** + Testing Library (jsdom) with an **80% coverage gate** (V8) in `pnpm check`.
  - **MSW** mocks the API at the network level, with handlers and fixtures typed by the generated OpenAPI types; unhandled requests fail the test.
  - One **Playwright** e2e in the shell against the real remote and catalogue-service.
- **Supply chain:** keep pnpm's minimum-release-age policy on, and let pnpm choose versions instead of forcing brand-new ones.

## Alternatives considered
- **Vite** — excellent dev experience, but Module Federation is a community plugin. Rspack implements webpack's federation natively, and MF 2.0's runtime and manifest come from the same team.
- **webpack** — the origin of Module Federation, but much slower than Rspack, with the same configuration model.
- **ESLint + Prettier** — the larger ecosystem, but two tools, two configs and slower runs. Biome covers what we need (hooks rules, a11y, formatting). This mirrors Spotless in the Java repos (ADR-029).
- **CSS Modules or CSS-in-JS** — also isolate styles. Tailwind gives a shared design vocabulary across apps without a shared library, and generates only the classes each app uses.
- **Redux / hand-written fetch + `useEffect`** — server data is a cache problem (staleness, dedupe, retries), which TanStack Query solves directly.
- **Hand-written API types** — drift silently from the service. Generating from the spec turns drift into a type error.
- **Jest** — slower and needs separate TypeScript/ESM setup. Vitest is Jest-compatible and native ESM.
- **Mocking `fetch` with `vi.fn`** — tests the mock, not the client. MSW lets the real client send real requests.

## Consequences
**Good**
- `pnpm check` = lint + typecheck + tests with coverage, the same gate shape as `./gradlew build`.
- A change to catalogue-service's API shows up as a type error after `pnpm generate:api`, in the client and in the test fixtures.
- Fast feedback: builds under a second, unit test runs about 2 s.

**Bad (costs we accept)**
- Many young tools moving fast (Rsbuild 2, MF 2.9, MSW 3, Vitest 5, Biome 2). We already hit an MSW option rename (`onUnhandledRequest` → `onUnhandledFrame`) and a Biome migration that switched all rules off. Upgrades need reading the changelog.
- Stuck on TypeScript 5.9 until `openapi-typescript` supports 7.
- The generated client only knows what the spec says. The service's own spec test (`OpenApiSpecTest`) keeps the spec honest.
- Tailwind markup is verbose, and the prefix (`dc:`) makes it more so.

## Interview one-liner
"Rsbuild because Module Federation is native in Rspack. API types generated from the service's OpenAPI spec, so contract drift is a compile error. TanStack Query because server data is a cache. And MSW so tests exercise the real HTTP client against a typed fake network. All behind one `pnpm check` with an 80% coverage gate."
