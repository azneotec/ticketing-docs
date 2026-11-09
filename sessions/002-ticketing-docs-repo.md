# Session 002 — `ticketing-docs` repo and ADRs 001–003
Phase: 1 · Estimated: 2–3 hours

## Goal
All planning and decision records live in a versioned Git repo, `ticketing-docs`, pushed to GitHub, with the first three ADRs written.

## Context to read
- `AGENTS.md`
- `docs/plan/2026-11-08-project-plan.md` (decisions, repo map, ADR backlog)
- `docs/plan/2026-11-09-decision-log.md` (rounds 1–4 and 9 — the reasoning for ADR-001..003)
- `docs/sessions/001-workstation-setup-handoff.md`

## Tasks
1. Create `ticketing-docs/` in the top-level folder; `git init` on `master`. Confirm Git identity is Azad Asanali <azneotech@gmail.com>.
2. Add `README.md` (purpose of the repo, folder map, how the elephant/goldfish loop works) and `.editorconfig` (UTF-8, LF, final newline, trim trailing whitespace, 2-space indent for md/yaml, 4 for java/kotlin).
3. **Move** `docs/plan`, `docs/sessions`, `docs/decisions` into `ticketing-docs/` (keeping the same sub-paths, e.g. `ticketing-docs/plan/...`). Show the exact move first and wait for approval. Remove the empty top-level `docs/` only after approval.
4. Update every path that changed in `AGENTS.md` (the start-of-session steps, the handoff location) and in `sessions/README.md`. `CLAUDE.md` stays as the single line `@AGENTS.md`.
5. Add `decisions/ADR-000-template.md` with sections: Status, Date, Context, Decision, Alternatives considered, Consequences (good and bad), Interview one-liner. Add `decisions/README.md` with the ADR index and the rule that accepted ADRs are never edited, only superseded.
6. Write, using the decision log as source (do not invent reasons):
   - `ADR-001-adr-format-and-location.md`
   - `ADR-002-event-ticketing-domain.md` (options A–D considered and why C)
   - `ADR-003-repo-per-service.md` (costs accepted: contracts repo, platform repo, Pact Broker, multi-PR changes)
   Keep each short and plain; Azad must be able to explain it aloud in 60 seconds.
7. First commit(s) using Conventional Commits (e.g. `docs: add ADR-001 ...`).
8. GitHub: **ask Azad whether the repo should be public or private** before creating it. Then create it (with `gh` if installed and logged in; otherwise give Azad the exact manual steps) and push `master`.

## Acceptance criteria
- `ticketing-docs` is a Git repo on `master`, pushed to GitHub, working tree clean.
- No `docs/` folder left at top level; all paths in `AGENTS.md` and `sessions/README.md` point to `ticketing-docs/`.
- ADR template, index and ADR-001..003 exist and match the decision log.
- This brief and its handoff are inside `ticketing-docs/sessions/`.

## Out of scope
- Other repos, any code, C4 diagrams, runbooks.
- CI on GitHub.

## Concepts to explain (2–4 lines each)
What an ADR is and why accepted ADRs are superseded rather than edited · Conventional Commits and how they enable semantic versioning · `.editorconfig` · trunk-based development as a solo developer · what `git push -u` and a remote are.
