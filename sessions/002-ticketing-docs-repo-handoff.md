# Handoff 002 — `ticketing-docs` repo and ADRs 001–003
Date: 2026-11-09

## Done
All acceptance criteria are met.
- `ticketing-docs/` is a Git repo on `master`, identity Azad Asanali <azneotech@gmail.com>.
- Pushed to a **public** GitHub repo: https://github.com/azneotec/ticketing-docs (`origin`, `master` tracks `origin/master`).
- `docs/plan`, `docs/sessions`, `docs/decisions` moved to `ticketing-docs/plan`, `ticketing-docs/sessions`, `ticketing-docs/decisions` (approved by Azad); empty top-level `docs/` removed.
- `AGENTS.md` paths now point to `ticketing-docs/...`; `sessions/README.md` names `ticketing-docs/sessions/` explicitly. `CLAUDE.md` unchanged (`@AGENTS.md`).
- ADR template, ADR index, ADR-001..003 written from the decision log.
- Commits (Conventional Commits):
  ```
  chore: init ticketing-docs with README and editorconfig
  docs: move plan and session briefs/handoffs into ticketing-docs
  docs: add ADR template and index
  docs: add ADR-001 ADR format and location
  docs: add ADR-002 event ticketing domain
  docs: add ADR-003 repo per service
  docs: add session 002 handoff
  ```

## Decisions made (and ADRs written)
- ADR-001, ADR-002, ADR-003 (all Accepted).
- **GitHub visibility: public** (Azad's choice).
- **ADR-002 option letters:** A order fulfilment, B travel booking, C event ticketing, D wallet. The decision log did not record the lettering; Azad approved this mapping.
- **Gaps in the decision log, not filled in:** the log only says "option A" for the docs home and does not record the rejected options, nor why order fulfilment / travel booking / wallet were rejected individually, nor why monorepo was rejected beyond "structured like an enterprise team". The ADRs state only what the log records. If the elephant remembers more, add it through a superseding ADR (or a typo-level fix now, before anyone relies on them).
- `.editorconfig` keeps trailing whitespace in Markdown (two trailing spaces = line break); all other files trim it.
- Briefs/handoff 001 and brief 002 were moved unchanged; the old `docs/...` paths inside them are history and were not rewritten.

## Files created or modified
In `ticketing-docs/` (repo):
- `README.md` (created)
- `.editorconfig` (created)
- `decisions/ADR-000-template.md` (created)
- `decisions/README.md` (created)
- `decisions/ADR-001-adr-format-and-location.md` (created)
- `decisions/ADR-002-event-ticketing-domain.md` (created)
- `decisions/ADR-003-repo-per-service.md` (created)
- `sessions/README.md` (moved, paths updated)
- `sessions/002-ticketing-docs-repo-handoff.md` (this file, created)
- `plan/2026-11-08-project-plan.md`, `plan/2026-11-09-decision-log.md`, `sessions/001-workstation-setup.md`, `sessions/001-workstation-setup-handoff.md`, `sessions/002-ticketing-docs-repo.md` (moved, unchanged)

In the top-level folder (not a repo):
- `AGENTS.md` (modified: `docs/` → `ticketing-docs/`; "in this repo" → "in this folder, mainly the `ticketing-docs/` repo")
- `docs/` (removed, was empty after the move)

Outside the project: new GitHub repo `azneotec/ticketing-docs`.

## Known issues / blockers
1. **`AGENTS.md` and `CLAUDE.md` are not versioned** — they sit in the top-level folder, outside any repo. Suggestion for the elephant: decide whether to keep a copy in `ticketing-docs` or make the top-level folder something else.
2. **GitHub account vs commit email:** the repo is under `azneotec`; commits use `azneotech@gmail.com`. If that email is not verified on the `azneotec` GitHub account, commits won't link to your profile. Check *GitHub → Settings → Emails*. Public repo means the email is visible in commit metadata.

## What I learned (for interview prep)
- **ADR and why superseded, not edited.** An Architecture Decision Record is a short note: the situation, what we chose, what we rejected, what it costs. It is a log of *why*. If you edit an accepted ADR, you erase the reasoning people relied on at the time; writing a new ADR that supersedes it keeps both the old and new reasoning and when it changed.
- **Conventional Commits → semantic versioning.** Commit messages start with a type: `fix:`, `feat:`, `docs:`, `chore:`, and `feat!:` / `BREAKING CHANGE:` for incompatible changes. Tools can read the history and pick the next version: `fix` → patch (1.0.1), `feat` → minor (1.1.0), breaking → major (2.0.0), and generate a changelog.
- **`.editorconfig`.** A file in the repo that every editor reads to apply the same encoding, line endings and indentation, so formatting rules live in Git instead of in one person's IDE settings.
- **Trunk-based development, solo.** Everyone (here: just you) commits small changes straight to one main branch (`master`) and keeps it releasable, instead of long-lived feature branches. Solo, it means: small commits, always green, short-lived branches only when a change is risky.
- **Remote and `git push -u`.** A remote is a named URL of another copy of the repo (`origin` = GitHub). `git push -u origin master` uploads `master` and sets it as the *upstream*, so later a plain `git push`/`git pull` knows where to go. `gh repo create --source . --push` did this for us.

## Suggested next step
Phase 1 step 3: `ticketing-build-conventions` (Java 25 toolchain, JaCoCo 80%, ArchUnit, versioning; publish to mavenLocal). On first opening it in IntelliJ, check the per-project Gradle settings noted in handoff 001.
