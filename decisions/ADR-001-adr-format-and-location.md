# ADR-001: ADR format and location

- **Status:** Accepted
- **Date:** 2026-11-09

## Context
This project exists for learning and for interview preparation. Every significant decision needs a record of *why*, in a form that is simple, readable and easy to talk through. Until session 002 the plan, the decision log and the session briefs/handoffs lived in an unversioned `docs/` folder in the top-level project folder. The project is also polyrepo (ADR-003), so there is no single code repo where cross-cutting decisions would naturally live.

## Decision
- Decisions are recorded as ADRs with the sections: Status, Date, Context, Decision, Alternatives considered, Consequences (good and bad), Interview one-liner. The template is `ADR-000-template.md`.
- ADRs live in the `ticketing-docs` repo under `decisions/`, next to the plan, the decision log and the session briefs/handoffs. Later this repo also holds C4 diagrams, runbooks and SLOs.
- Accepted ADRs are never edited; a change of mind is a new ADR that supersedes the old one.
- The top-level folder keeps only `CLAUDE.md` and an `AGENTS.md` symlink into `ticketing-docs` (since session 003).

## Alternatives considered
- **A. Everything in the `ticketing-docs` repo, versioned** — chosen.
- **B. Plan and session briefs/handoffs stay in the unversioned top-level folder** (the state before session 002) — rejected: no history and no backup, no review as diffs, not published to GitHub.

## Consequences
**Good**
- Every plan change and decision has Git history and is pushed to GitHub with the rest of the project.
- One place for humans and agents to find the plan, briefs, handoffs and decisions.
- The interview one-liner forces each decision to be explainable in a sentence.

**Bad (costs we accept)**
- `CLAUDE.md` and the `AGENTS.md` symlink must sit in the top-level folder, outside any repo, because agents start there. Since session 003 the real `AGENTS.md` lives in `ticketing-docs` (versioned) and the top-level file is a relative symlink to it.
- A decision and the code that implements it land in different repos, so they are separate commits.
- Superseding instead of editing means more files over time; the index in `decisions/README.md` must stay current.

## Interview one-liner
"We keep short ADRs, with context, decision, alternatives, consequences and a one-line summary, in a dedicated docs repo, and we supersede them instead of editing, so the history of *why* is never lost."
