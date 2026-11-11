# Session 009 — Front-end monorepo `ticketing-web` (before Phase 2)
Phase: 1 → 2 transition · Estimated: 2–3 hours

## Goal
Merge `web-shell` and `web-discover` into one pnpm-workspace repo, `ticketing-web`, keeping their Git history and tags, with tooling that stops apps importing each other, a type-checked shell ↔ remote contract, and independent builds per app. Back-end services stay one repo each.

## Context to read
- `AGENTS.md`
- `ticketing-docs/decisions/ADR-003-*.md`, `ADR-019-*.md`, `ADR-035-*.md`
- `ticketing-docs/sessions/008-web-shell-discover-handoff.md` (known issues, esp. #2)
- `ticketing-docs/plan/2026-11-12-phase-1-retro.md`
- READMEs of `web-shell` and `web-discover`

## Decision (made by Azad, reviewed by the elephant — record it; do not re-decide)
- **Front end: monorepo `ticketing-web`; back-end services: polyrepo (unchanged).**
- For: shared singletons (React, Router, TanStack Query) upgrade in one PR; shared contract package type-checks shell ↔ remote; one Biome/tsconfig/Vitest config, one lockfile, one CI pipeline later; e2e next to both apps. Independent deploys remain because Module Federation independence comes from separate builds and runtime loading, not separate repos.
- Against (accepted): breaks symmetry with the back end; risk of build-time imports between apps (mitigated by tooling); per-app tags; weaker "separate repos, separate teams" lesson.
- Interview framing: polyrepo where ownership and release cycles differ (services), monorepo where runtime-shared dependencies must move together (front end).

## Tasks
### A — docs (`ticketing-docs`)
1. Decision log **Round 16** (the decision above, with for/against).
2. **ADR-037** "Front-end monorepo; ADR-003 narrowed to back-end services". Status Accepted; *Partially supersedes ADR-003; amends ADR-019.*
3. Add a status note line to **ADR-003** ("Partially superseded by ADR-037 for front-end apps") and **ADR-019** ("Repository layout amended by ADR-037"). Change nothing else in them.
4. Plan: repo map — replace `web-shell`, `web-discover`, `web-checkout`, `web-account` rows with one `ticketing-web` row (`apps/shell`, `apps/discover`; later `apps/checkout`, `apps/account`); ADR backlog adds 037. Index updated.

### B — migration (history kept)
5. Show Azad the full plan of git commands first and wait for approval. Approach: clone each repo to a temp area, `git filter-repo --to-subdirectory-filter apps/shell` (resp. `apps/discover`) (install `git-filter-repo` via Homebrew with approval), then in a new `ticketing-web` repo merge both with `--allow-unrelated-histories`. Recreate the unpushed tags on the rewritten commits as **`shell-v0.1.0`** and **`discover-v0.1.0`** (annotated). Verify `git log --follow` on a moved file shows the original commits.
6. Workspace setup: root `package.json` (`packageManager` pinned), `pnpm-workspace.yaml` (`apps/*`, `packages/*`), one `pnpm-lock.yaml`, `.nvmrc`, `.editorconfig`, `.gitattributes`, `.gitignore` (node_modules, dist, coverage, playwright reports), root README.
7. `packages/config`: shared Biome config, base `tsconfig`, Vitest base config (80% gate); apps extend them; remove the per-app duplicates.
8. `packages/discover-contract`: exposed module names and props types (e.g. `EventListProps`, `EventDetailProps`) and the remote name; Discover's exposed components and the shell's loader both use it; **type-only imports** (`import type`). Turns known issue #2 into compile-time checks.
9. **Boundary enforcement**: apps may import `packages/*` only, never another app, and the contract package only as types. Use Biome `noRestrictedImports` and/or `dependency-cruiser`; add a check that a deliberately bad import fails `pnpm check` (proof script or test).
10. Independent builds: `pnpm --filter shell build` and `pnpm --filter discover build` work alone and produce separate `dist/` outputs; root scripts `dev` (both), `check` (all), `e2e`.
11. Per-app release steps in the README (tag format `<app>-vX.Y.Z`; `package.json` version per app).

### C — verify
12. `pnpm check` green at root (lint, typecheck, tests, 80% coverage per app). Playwright e2e passes (catalogue-service + `make up PROFILES="core"` running). Browser → catalogue trace still appears in Tempo (Azad confirms quickly).
13. Old folders: show the change, then with approval move `web-shell/` and `web-discover/` to `_archive/` at the top level (do not delete). Update `AGENTS.md` if any paths mention them.
14. Conventional Commits; **Azad runs** `gh repo create azneotec/ticketing-web --public --source . --push` and `git push origin shell-v0.1.0 discover-v0.1.0`.

## Acceptance criteria
- `ticketing-web` contains both apps with their original history and tags `shell-v0.1.0`, `discover-v0.1.0`; pushed to GitHub.
- A cross-app import fails the check; contract mismatches fail typecheck.
- Each app builds independently; root `pnpm check` and e2e pass; trace still visible.
- Round 16, ADR-037, status lines on ADR-003/019, plan repo map updated; old folders archived; all repos clean.

## Out of scope
- New features, Checkout/Account apps, CI pipeline, Nx/Turborepo (note in handoff if worth considering later).

## Concepts to explain (2–4 lines each)
Monorepo vs polyrepo trade-offs · pnpm workspaces and `workspace:*` · rewriting history with `git filter-repo` and why it's safe here (nothing pushed) · enforcing module boundaries · type-only imports and why they don't couple runtimes · independent build/deploy inside a monorepo.
