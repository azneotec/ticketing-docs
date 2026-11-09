# ADR-029: Formatting enforced by the build (Spotless + google-java-format)

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning round 10; built in session 003)

## Context
Code is written by Azad in IntelliJ and by several fresh agent sessions (Claude Code, Codex) across about ten repos. If every author formats differently, diffs fill up with whitespace and import-order noise, and reviews (and ADR-worthy changes) get harder to read. Round 10 decided to enforce formatting from the first line of code.

## Decision
- Every Java module gets **Spotless** with **google-java-format** (2.0.0) through `ticketing.java-conventions`. It also removes unused imports, trims trailing whitespace and ends files with a newline.
- `spotlessCheck` is part of `check`, so a badly formatted file fails `./gradlew build`. `./gradlew spotlessApply` fixes it.
- The formatter version is pinned in the conventions plugin; changing it is a conventions release.

## Alternatives considered
- **IDE formatter settings only** — live on one machine, not in Git. Agents running Gradle from a terminal never see them, and nothing fails when they are ignored.
- **Checkstyle** — reports style violations but cannot fix them, and needs a long rule file to maintain. A formatter that rewrites the code removes the discussion entirely.
- **Palantir Java Format** (a google-java-format fork with 120-column lines) — reasonable, but google-java-format is the more widely known default, and the choice matters less than having exactly one.

## Consequences
**Good**
- Zero formatting noise in diffs; every repo looks the same.
- No style debates: the tool decides.
- Agents and humans get the same answer from the same command.

**Bad (costs we accept)**
- google-java-format is not configurable (2-space indent, 100 columns), and it differs from IntelliJ's default style. Install the google-java-format IntelliJ plugin, or run `spotlessApply` before committing.
- A formatter upgrade can reformat many files at once; do it as a separate `style:` commit.
- `spotlessCheck` adds a few seconds to each build.

## Interview one-liner
"Formatting is enforced in the build, not the IDE: Spotless with google-java-format fails `check` on any unformatted file, so humans and AI agents produce identical diffs and we never discuss style in review."
