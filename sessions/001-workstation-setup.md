# Session 001 — Workstation setup
Phase: 1 · Estimated: 2–3 hours

## Goal
A verified Mac workstation with every Phase 1 tool installed, and Azad understanding what each tool is for.

## Context to read
- `CLAUDE.md`
- `docs/plan/2026-11-08-project-plan.md` (Phase 1 only)

## Tasks
1. Check what is already installed (`java -version`, `gradle -v`, `docker version`, `node -v`, `git --version`, `brew -v`, `claude --version`). Report before installing anything.
2. Install missing tools, preferring Homebrew and SDKMAN:
   - Homebrew
   - SDKMAN, then JDK 25 (Temurin) and Gradle (latest)
   - OrbStack (already chosen) — confirm `docker` and `docker compose` work
   - Node LTS (via `fnm` or `nvm`) and `pnpm`
   - Git, with `user.name` / `user.email` set and default branch `master`
   - `kubectl`, `helm`, `minikube` (install only; no cluster yet)
   - `grpcurl` and `jq` (handy later)
3. Configure OrbStack for a 16 GB Mac: set its memory limit to about 8 GB (OrbStack Settings → System, or `orb config set memory_mib 8192`) and confirm the setting. Explain why the cap matters.
4. Verify: run `docker run --rm hello-world`; print versions of every tool into the handoff.
5. Recommend IntelliJ IDEA settings for Java 25 + Gradle, including a sensible IDE heap for a 16 GB machine (Azad installs/configures himself).

## Acceptance criteria
- Every tool above prints a version.
- `docker run hello-world` succeeds via OrbStack.
- OrbStack memory limit is ~8 GB.
- Handoff lists versions and any manual steps Azad must still do.

## Out of scope
- Creating any repo or project code.
- Starting minikube.

## Concepts to explain (2–4 lines each)
SDKMAN vs Homebrew for JDKs · what OrbStack replaces · what `docker compose` is · why kubectl/helm/minikube are separate tools · what pnpm adds over npm.
