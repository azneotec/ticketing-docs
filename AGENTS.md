# Distributed Ticketing Platform — agent rules

Single source of truth for all coding agents (Claude Code reads it via `CLAUDE.md`, Codex reads it directly). Change rules here only.
This file lives in the `ticketing-docs` repo (versioned); the top-level `AGENTS.md` is a relative symlink to it.

You are a **goldfish** build session. You have no memory of earlier sessions.
Everything you need is in this folder, mainly the `ticketing-docs/` repo. Read before you act.

## Start of every session
1. Read this file.
2. Read `ticketing-docs/plan/2026-11-08-project-plan.md` (decisions; do not re-decide them).
3. Skim `ticketing-docs/plan/2026-11-09-decision-log.md` when you need the reasoning behind a decision.
4. Read the session brief you were given (`ticketing-docs/sessions/NNN-*.md`).
5. Read the latest handoff note in `ticketing-docs/sessions/` for current state.
6. Write a short plan for the brief and **wait for Azad's approval** before changing anything.

## During the session
- Do only what the brief asks. Anything else goes in the handoff as a suggestion.
- After each major step, summarise what you did and what's next.
- Before deleting, overwriting or renaming any existing file, show the change and wait for confirmation.
- Never touch files outside this folder, except tool installs or global config that the brief explicitly asks for.
- Explain each new tool or concept briefly (what it is, why we use it). Azad is learning these from the ground up.
- Blocker rule: time-box to about 3 hours of effort. If stuck, record it in the handoff as a known issue and move on.

## End of every session
- Write `ticketing-docs/sessions/NNN-<task>-handoff.md` using the template in `ticketing-docs/sessions/README.md`.
- List every file you created or modified, with its path.

## Engineering rules
- Java 25, latest stable Spring Boot, Gradle Kotlin DSL.
- One Git repo per service, as subfolders of this folder (see plan's repo map). Trunk-based on `master`, semantic versioning.
- Strict hexagonal: `domain` module has zero framework dependencies; enforce with ArchUnit.
- Test coverage gate: 80% per service (JaCoCo). Testcontainers for adapters.
- Use modern Java features deliberately (records, sealed interfaces, pattern matching, virtual threads) and record why in an ADR.
- Observability from day one: OpenTelemetry traces, metrics, logs.
- Every significant decision gets an ADR in `ticketing-docs/decisions/` (format: context, decision, alternatives, consequences, interview one-liner).

## File naming
- New docs: `YYYY-MM-DD-descriptive-name.md`, except session files (`NNN-task.md`) and ADRs (`ADR-NNN-title.md`).

## Local environment
- MacBook M1, **16 GB RAM**. Docker runtime: OrbStack, capped at ~8 GB.
- Resource rules: run only what the current task needs; never Compose and minikube together; set memory limits on every container; Spring services `-Xmx256m`–`384m`; one shared Postgres container with a database per service; observability via `grafana/otel-lgtm`. See plan rows "Hardware" and "Resource budget".
- Claude Code and Codex installed. Git identity: Azad Asanali <azneotech@gmail.com>. Kubernetes later: minikube + Helm.
- Project folder: `~/learn/java_workspace/distributed-ticketing`.
