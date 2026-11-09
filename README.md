# ticketing-docs

Home for every planning and decision record of the **Distributed Ticketing Platform**, a learning project: an event ticketing system built as microservices (Java 25, Spring Boot, Kafka, gRPC, Postgres, Kubernetes), finished to senior-engineer standard.

Keeping these docs in Git means every plan change, brief and decision has history, can be reviewed as a diff, and is published alongside the code repos.

## Folder map

| Folder | What lives there |
|---|---|
| `plan/` | Project plan (source of truth for decisions) and the decision log (the reasoning behind them) |
| `sessions/` | Session briefs (`NNN-task.md`) and handoff notes (`NNN-task-handoff.md`), plus the templates in `sessions/README.md` |
| `decisions/` | Architecture Decision Records (`ADR-NNN-title.md`), the template `ADR-000-template.md` and the index in `decisions/README.md` |

Later: C4 diagrams, runbooks and SLOs.

## How the work loop runs (elephant and goldfish)

- **Elephant**: a long-running planning chat. It keeps the long memory, writes one session brief at a time, and reviews handoffs. It writes no code.
- **Goldfish**: a fresh Claude Code (or Codex) session started in the top-level project folder. It has no memory: it reads `AGENTS.md`, the plan, its brief and the latest handoff, proposes a plan, waits for approval, does the one brief, and writes a handoff.

1. Elephant writes `sessions/NNN-<task>.md`.
2. Goldfish is told "Do session brief NNN", plans, gets approval, builds.
3. Goldfish writes `sessions/NNN-<task>-handoff.md`.
4. Elephant reviews the handoff and writes the next brief.

Agent rules live in `AGENTS.md` in the top-level folder (outside this repo); `CLAUDE.md` there only imports it.

## Conventions

- File names: `YYYY-MM-DD-descriptive-name.md`, except session files (`NNN-task.md`) and ADRs (`ADR-NNN-title.md`).
- Commits: [Conventional Commits](https://www.conventionalcommits.org/) (`docs: ...`, `chore: ...`).
- Branching: trunk-based on `master`.
- Accepted ADRs are never edited; they are superseded by a new ADR (see `decisions/README.md`).
