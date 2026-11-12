# Handoff 009 — Front-end monorepo `ticketing-web`
Date: 2026-11-12

## Done
**A — docs (`ticketing-docs` `aa28d26`)**
- Decision log **Round 16**: the decision, with the for and against points, the interview framing and how history was migrated.
- **ADR-037** "Front-end monorepo; ADR-003 narrowed to back-end services". Accepted; partially supersedes ADR-003, amends ADR-019.
- Status-note lines added to ADR-003 and ADR-019 (nothing else changed). ADR index updated, with 003 marked "partially superseded by 037".
- Plan:
  - Repo map: the four `web-*` rows became one `ticketing-web` row.
  - ADR backlog: 037 added.
  - "Repos" decision row: now says polyrepo for back-end services and a monorepo for the front end, with per-app tags. This row isn't named in the brief; I changed it so the plan doesn't contradict itself.

**B — migration with history (`ticketing-web`, 7 commits on top of the imported history)**
- Installed `git-filter-repo` 2.47.0 with Homebrew.
- Cloned each repo to the scratchpad, ran `git filter-repo --to-subdirectory-filter apps/<app>`, then merged both into a new repo with `--allow-unrelated-histories`.
- Tags recreated as annotated **`shell-v0.1.0`** and **`discover-v0.1.0`**, on the rewritten release commits. The old `v0.1.0` tags were removed from the rewritten clones.
- Checked with `git log --follow apps/shell/package.json`: all 5 original shell commits show (and all of Discover's for its files).
- **Workspace root:**
  - `package.json` pins `packageManager: pnpm@12.18.0`. Scripts: `dev` (both apps), `build`, `lint`, `format`, `typecheck`, `boundaries`, `boundaries:proof`, `test`, `check`, `e2e`.
  - Also at the root: `pnpm-workspace.yaml` (`apps/*`, `packages/*`), one `pnpm-lock.yaml`, `.nvmrc`, `.editorconfig`, `.gitattributes`, `.gitignore`, `biome.json`, README.
  - Per-app duplicates of these files were removed.
- **Dependency versions:** resolved to the same direct versions as before (React 19.3.1, OTel 2.12.1/0.223.1, Biome 2.6.6, TS 5.9.3, …).
- **Apps renamed** to the packages `shell` and `discover`, so `pnpm --filter shell …` works. **Runtime names unchanged:** Module Federation `web_shell`/`web_discover`, and OTel `service.name=web-shell`. The Tempo query still works.
- **`packages/config` (`@ticketing/config`):**
  - `biome.base.json`, `tsconfig.base.json`, and `vitest.base.ts` with the 80% gate.
  - Apps use `extends` for tsconfig and `mergeConfig` for Vitest. The root `biome.json` extends the shared one.
  - The gate was proved still active: setting it to 99% failed the shell's tests, then I set it back.
- **`packages/discover-contract` (`@ticketing/discover-contract`), types only:**
  - `DiscoverRemoteName`, `EventListProps`, `EventDetailProps`, `DiscoverExposes` (`'./EventList'` → props) and the derived `DiscoverModules`/`DiscoverModuleId` (`'web_discover/EventList'` → props).
  - **Discover** checks itself against it:
    - `name: 'web_discover' satisfies DiscoverRemoteName`
    - `exposes: {...} satisfies Record<keyof DiscoverExposes, string>`
    - each exposed default export `satisfies ComponentType<…Props>`
  - **Shell:**
    - New `DiscoverSlot<M extends DiscoverModuleId>` (pages use it) and `DISCOVER satisfies DiscoverRemoteName`.
    - `DiscoverSlot.test.tsx` has 4 `@ts-expect-error` cases (unknown module, missing `id`, wrong `id` type, props on `EventList`). Typecheck fails if any of them stops being an error.
  - **Broken on purpose, each a type error:** a renamed expose key, an `EventDetail` taking `{ id: number }`, and a new required prop in the contract (the error showed in the shell's pages).
- **Boundaries (dependency-cruiser 18.7.2, `.dependency-cruiser.cjs`):**
  - The rules: `no-cross-app-imports`, `contracts-are-type-only`, `packages-do-not-import-apps`, `no-unresolvable-imports`. `tsPreCompilationDeps` is on, so type-only imports are seen; I checked that all 5 contract edges are marked `type-only`.
  - `scripts/boundaries-proof.mjs` plants one bad import per rule (a relative import into Discover, a value import of the contract, `import … from 'discover'`, and a package importing an app). It asserts each rule fails, and always deletes the planted file (which is also git-ignored).
  - Both the rules and the proof run in `pnpm check`.
- **Paths fixed:**
  - Discover's `generate:api` now reads `../../../catalogue-service/docs/openapi.yaml`.
  - Playwright starts Discover with `pnpm --filter discover dev`.
- **READMEs:**
  - Root: layout, commands, rules between apps, independent builds and deploys, **releasing an app** (`<app>-vX.Y.Z`, own `package.json` version, `chore(<app>): release X.Y.Z`), and the history note.
  - App READMEs updated for the monorepo; their release sections point to the root README.

**C — verify**
- Root `pnpm check` green:
  - Biome
  - boundaries, with 4/4 proof cases caught
  - typecheck for both apps and the contract
  - tests: Discover 29 (lines 94.1 / branches 97.2), shell **16** (lines 85.7 / branches 83.3). The shell's 14 old tests plus 2 new ones in `DiscoverSlot.test.tsx`.
- `pnpm --filter shell build` and `pnpm --filter discover build` each work alone, into separate `apps/<app>/dist/`. Checked: Discover's code isn't in the shell's bundle, only the module id string.
- `pnpm e2e` from the root passes (4.2 s), with `make up PROFILES="core"` and catalogue-service running.
- Browser trace in Tempo: `c252a73760676cf5f3f351da406d5d5c`, going `web-shell GET` (33 ms) → `http get /events/{id}` (23 ms) → `GetEvent` → `connection`/`query`. **Confirmed by Azad.**
- `web-shell/` and `web-discover/` moved to `_archive/` (with approval). `AGENTS.md` didn't mention them.
- Platform, catalogue-service and dev servers stopped afterwards (`make down`; volumes kept).

## Azad: create the GitHub repo and push
Auto mode blocks `gh repo create` for me. Run:
```bash
cd ~/learn/java_workspace/distributed-ticketing/ticketing-web && gh repo create azneotec/ticketing-web --public --source . --push
```
```bash
cd ~/learn/java_workspace/distributed-ticketing/ticketing-web && git push origin shell-v0.1.0 discover-v0.1.0
```
Then push `ticketing-docs` (this handoff and `aa28d26` are local commits):
```bash
cd ~/learn/java_workspace/distributed-ticketing/ticketing-docs && git push
```

## Decisions made (and ADRs written)
- ADR-037 (above). Smaller choices:
  - **dependency-cruiser over Biome `noRestrictedImports`:** Biome's rule can't tell `import type` from a value import, so it can't enforce "contract as types only". dependency-cruiser sees both, and also catches imports by path and by package name.
  - **Contract as types only, including the names.** Module ids and the remote name are string-literal types. Runtime strings stay in each app and are checked with `satisfies`, so the package has no runtime code at all.
  - **`DiscoverSlot` wraps the generic `RemoteSlot`.** `RemoteSlot` stays untyped by module, so its tests (and later remotes) don't depend on Discover's contract.
  - **Shared Biome config named `biome.base.json`:** a nested `biome.json` makes Biome treat it as a second root config and refuse to run.
  - **Root `@ticketing/config` dev dependency:** the root `biome.json` resolves `extends` from the root's own `node_modules`.
  - **Proof script in `pnpm check`** (not a one-off), so a rule that silently stops matching (for example after a path change) fails the build. It adds about 4 s.
  - **The contract's `version` is `0.1.0` but it isn't tagged.** It ships inside the apps that use it, not on its own.

## Files created or modified
`ticketing-docs/`:
- `plan/2026-11-09-decision-log.md` (Round 16), `plan/2026-11-08-project-plan.md` (Repos row, repo map, ADR backlog)
- `decisions/ADR-037-frontend-monorepo.md` (new), `decisions/ADR-003-repo-per-service.md` and `decisions/ADR-019-module-federation-split.md` (status note line only), `decisions/README.md` (index)
- `sessions/009-ticketing-web-monorepo.md` (brief, committed), `sessions/009-ticketing-web-monorepo-handoff.md` (this file)

`ticketing-web/` (new repo):
- Root: `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `biome.json`, `.dependency-cruiser.cjs`, `README.md`, `scripts/boundaries-proof.mjs`. These moved from the apps: `.editorconfig`, `.gitattributes`, `.gitignore` (plus a proof-file entry), `.nvmrc`.
- `packages/config/`: `package.json`, `biome.base.json`, `tsconfig.base.json`, `vitest.base.ts`
- `packages/discover-contract/`: `package.json`, `tsconfig.json`, `src/index.ts`
- `apps/shell/` (modified): `package.json`, `tsconfig.json`, `vitest.config.ts`, `playwright.config.ts`, `README.md`, `src/pages/pages.tsx`, `src/remotes/registry.ts`. New: `src/remotes/DiscoverSlot.tsx`, `src/remotes/DiscoverSlot.test.tsx`. Removed: `biome.json`, `pnpm-lock.yaml`.
- `apps/discover/` (modified): `package.json`, `tsconfig.json`, `vitest.config.ts`, `rsbuild.config.ts`, `README.md`, `src/exposed/EventList.tsx`, `src/exposed/EventDetail.tsx`. Removed: `biome.json`, `pnpm-lock.yaml`.

Outside the repos:
- `_archive/web-shell`, `_archive/web-discover` (moved, untouched).
- Homebrew: `git-filter-repo` 2.47.0.
- Scratchpad: the rewritten temp clones (session-only).
- New traces in the `otel-lgtm` volume.

## Known issues / blockers
1. ~~`ticketing-web` isn't on GitHub yet.~~ **Pushed by Azad (2026-11-12):** `azneotec/ticketing-web` `master`, tags `shell-v0.1.0` and `discover-v0.1.0`; `ticketing-docs` pushed too.
2. My own mistake, fixed: while breaking the contract on purpose, `git checkout` on an uncommitted file also undid my contract edit in Discover's `rsbuild.config.ts`. I re-applied it before committing, and the full check passed. Lesson: commit before deliberate breaks.
3. The contract is checked at compile time only. A Discover **deployed** from an older commit can still disagree with a newer shell at runtime. Release order (remote first for breaking changes) is in the root README, not enforced.
4. `dist/` folders and `node_modules` exist locally in `ticketing-web` (git-ignored).
5. Carried over: `PUBLIC_*` baked in at build time; `/otlp` dev-server-only; TS 5.9 and OTel JS 2.x pins; StrictMode double spans; no CI. The full list is in the Phase 1 retro.

## What I learned (for interview prep)
- **Monorepo vs polyrepo.**
  - Polyrepo gives hard boundaries, independent versions and access control per repo. It costs multi-PR changes, version drift and duplicated tooling.
  - Monorepo gives atomic cross-project changes, one lockfile and one config. It costs needing tooling to keep boundaries, plus CI and versioning that must understand "which project changed".
  - Rule of thumb: split where ownership and release cycles differ, join where things must change together.
- **pnpm workspaces and `workspace:*`.**
  - `pnpm-workspace.yaml` lists the package folders, and one install links them all.
  - `workspace:*` means "use the local package". pnpm symlinks it into `node_modules` (on publish it would be replaced with the real version).
  - `--filter <name>` runs a script in one package. pnpm's strict `node_modules` means a package only sees what it declares.
- **Rewriting history with `git filter-repo`.**
  - It rewrites every commit (here, moving all paths under `apps/<app>/`), so every commit id changes.
  - Safe only when nobody has the old history: nothing was pushed, and we worked on clones.
  - `git merge --allow-unrelated-histories` joins two root commits, so both histories live on in one repo.
- **Enforcing module boundaries.**
  - In one repo, any file can import any other, and the bundler happily inlines it. That would turn runtime composition into build-time coupling without anyone noticing.
  - A dependency rule checker (dependency-cruiser, Nx's `enforce-module-boundaries`) makes the architecture a failing build, like ArchUnit in the Java services.
  - The proof script is how we test the tests.
- **Type-only imports.**
  - `import type` is removed entirely by the compiler (`verbatimModuleSyntax` guarantees it). So the shell's bundle contains no Discover code, yet `tsc` checks both sides against one declaration.
  - Compile-time agreement, runtime independence.
- **Independent build and deploy inside a monorepo.**
  - Deploy independence comes from separate build outputs and runtime loading (manifest URL), not from repo boundaries.
  - Each app still has its own `dist/`, version and tag (`shell-v0.1.0`). Tools like Nx or Turborepo add "affected" builds and caching when a repo grows; with two apps, `pnpm --filter` is enough.

## Suggested next step
- Azad: run the `gh repo create` and push commands above.
- Elephant: the Phase 2 kickoff. Retro point 6 (CI) now has a simpler front-end half: one GitHub Actions workflow running `pnpm check` for `ticketing-web`. Nx/Turborepo is worth reconsidering only when Checkout arrives, if CI time grows.
