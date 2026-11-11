# ADR-037: Front-end monorepo; ADR-003 narrowed to back-end services

- **Status:** Accepted. Partially supersedes ADR-003 (front-end apps only); amends ADR-019 (repository layout).
- **Date:** 2026-11-12 (decided in round 16; built in session 009)

## Context
ADR-003 put every service and every front-end app in its own repo. After Phase 1 the two web apps (`web-shell`, `web-discover`) showed the cost of that on the front end:
- React, React Router and TanStack Query are **runtime singletons** shared by every app (ADR-019). Upgrading one means upgrading all of them together, which across repos is one PR per app plus a careful order.
- The shell ↔ remote contract (exposed module names and their props) could not be type-checked across repos; only the e2e test caught a mismatch (handoff 008, known issue #2).
- Each app carried its own copy of the Biome, TypeScript and Vitest config and its own lockfile.

The back-end services don't have this problem: they share contracts as versioned artifacts and run as separate processes with separate dependencies.

## Decision
- **One repo `ticketing-web` for all front-end apps**, as a pnpm workspace:
  - `apps/shell`, `apps/discover` (later `apps/checkout`, `apps/account`).
  - `packages/config`: shared Biome, TypeScript and Vitest (80% gate) config.
  - `packages/discover-contract`: Discover's remote name, exposed module ids and props types.
- **Back-end services stay polyrepo** (ADR-003 still applies to them).
- **Boundaries enforced by the build:** an app may import `packages/*` only, never another app; contract packages may only be imported with `import type`. `dependency-cruiser` checks this in `pnpm check`.
- **Independent builds and deploys stay:** each app builds alone (`pnpm --filter shell build`) into its own `dist/`, has its own `package.json` version and is tagged `<app>-vX.Y.Z`. The shell still loads remotes at runtime from configured URLs (ADR-019).
- **History kept:** both repos were moved in with `git filter-repo`, so `git log --follow` shows their original commits.

## Alternatives considered
- **Keep polyrepo for the front end (ADR-003 as is)** — realistic for separate teams, but every shared-singleton upgrade is a multi-repo change, and the contract stays untyped. MF's own type sharing (`dts`) would type it, but needs the remote running at build time.
- **One monorepo for everything, back end included** — would hide the versioning and contract problems ADR-003 wants to experience, and services don't share runtime dependencies.
- **Publish the contract as an npm package from the remote's repo** — types across repos, but adds a registry and a publish step for every props change.
- **Nx or Turborepo** — task caching and affected-only builds. Not needed for two small apps; pnpm's `--filter` is enough. Revisit if builds get slow or CI needs "affected" runs.

## Consequences
**Good**
- One PR upgrades React (or any shared singleton) in every app, with one lockfile.
- A wrong module id or wrong props between shell and Discover is a compile error.
- One config for lint, types and tests; e2e lives next to both apps.

**Bad (costs we accept)**
- Breaks the symmetry with the back end, which has to be explained.
- An app could import another app's code at build time, coupling their deploys. Only the boundary check prevents it.
- Release tags are per app (`shell-v0.1.0`), not per repo, and versioning tools must understand that.
- Weaker "separate repos, separate teams" lesson on the front end.

## Interview one-liner
"Polyrepo where ownership and release cycles differ, which is the services; monorepo where runtime-shared dependencies must move together, which is the micro-frontends. They still deploy independently, because Module Federation's independence comes from separate builds and runtime loading, not separate repos. A boundary check stops apps importing each other, and a type-only contract package type-checks the shell against the remote."
