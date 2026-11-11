# ADR-019: Micro-frontends with Module Federation: shell + Discover, Checkout, Account

- **Status:** Accepted
- **Date:** 2026-11-12 (decided in planning; built in session 008)

## Context
The plan calls for a React web front end split like the back end: one team-sized piece per business capability (browse, buy, manage account), each deployable on its own. Part of the goal is learning what micro-frontends really cost, not only what they promise. We need a split, a way to compose the pieces in the browser, and a rule for what is shared and what happens when a piece fails.

## Decision
- **One host, `web-shell`**, owns the page frame (layout, navigation), routing, global CSS reset, the TanStack Query client, and browser tracing.
- **Remotes per capability:**
  - `web-discover`: browse and view events (Phase 1).
  - `web-checkout`: Phase 2.
  - `web-account`: Phase 4.

  Each is its own repo, its own dev server and deploy, and also runs standalone.
- **Runtime integration with Module Federation 2.0.** The shell has no remotes in its build. At start-up it registers each remote's manifest (`<url>/mf-manifest.json`) from configuration (`PUBLIC_DISCOVER_URL`) and loads modules with `loadRemote` inside `React.lazy`.
- **Contract = exposed modules + props.** Discover exposes `./EventList` (no props) and `./EventDetail` (`{ id }`). The host owns URLs; remotes only link to agreed paths (`/events/:id`).
- **Shared as singletons:** `react`, `react-dom`, `react-router`, `@tanstack/react-query`. The host provides them, and remotes reuse the host's copy (`singleton: true`, `requiredVersion` from `package.json`).
  - Loaded lazily behind an async boundary (`import('./bootstrap')`).
  - Nothing else is shared: each app bundles its own copies of other libraries.
- **Failure isolation:** every remote renders inside an error boundary. If a remote can't be loaded or throws, only its slot shows "This section is unavailable" with **Try again**. Header and navigation keep working.
- **Styles don't leak:** the shell owns the reset. Remotes build their own Tailwind CSS with a prefix (`dc:`) and no reset.

## Alternatives considered
- **One React app (modular monolith)** — simplest, fastest first load, one dependency tree. Rejected for this project because independent deployment of front-end pieces is a learning goal. It would be the right default for a small real team.
- **Build-time composition (remotes as npm packages)** — type-safe and simple, but every remote change means rebuilding and redeploying the shell: no independent deploys.
- **iframes** — perfect isolation, but poor UX (routing, sizing, focus, shared state) and duplicated runtimes.
- **Remote URLs hard-coded in the shell's build config** — simpler, but moving or versioning a remote would mean a shell rebuild. Runtime registration costs a few lines.
- **Sharing a UI component library now** — premature with one remote. A written design convention (colours, spacing) is enough until Checkout exists.

## Consequences
**Good**
- Discover can be developed, tested and deployed without touching the shell, which only knows its URL and module names.
- A broken or unreachable remote degrades one area of the page, not the whole site (checked by stopping Discover).
- One React, router and query cache for the page: remotes join the shell's routing and caching.

**Bad (costs we accept)**
- Runtime coupling through shared singletons: upgrading React or React Router is a coordinated change across all apps. A mismatch is only a console warning.
- The contract between shell and remote (module names, props) is not type-checked across repos (MF type sharing is off). It is covered by the e2e test.
- More repos, servers and pipelines, a slower first load (manifest + remote chunks), and duplicated code and dependencies across apps.
- Remote URLs are baked in at build time for now. Runtime config for Kubernetes is Phase 4.

## Interview one-liner
"The shell owns layout, routing and shared singletons like React and the query cache. Feature remotes are loaded at runtime from configured manifest URLs, so each deploys independently, and each sits behind an error boundary, so one failing remote can't take down the page. The price is coordinated upgrades of the shared libraries and a cross-repo contract we test end to end."
