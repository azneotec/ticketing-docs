# Sessions: elephant and goldfish

- **Elephant** = the planning conversation in Claude (claude.ai). Holds long memory, writes session briefs, reviews handoffs. Writes no code.
- **Goldfish** = a fresh Claude Code session in the top-level project folder (`~/learn/java_workspace/distributed-ticketing`). Reads the repo, does one brief, writes a handoff, ends.

## Loop
1. Elephant writes `ticketing-docs/sessions/NNN-<task>.md`.
2. In a terminal: `cd ~/learn/java_workspace/distributed-ticketing`, run `claude`, and say: "Do session brief NNN."
3. Goldfish plans → you approve → it builds, tests, writes `ticketing-docs/sessions/NNN-<task>-handoff.md`.
4. Back in the elephant chat: "Handoff NNN is ready." Elephant reviews and writes the next brief.

Start each goldfish with a fresh session (`/clear` or a new `claude`), never a long-running one.

## Brief template
```
# Session NNN — <task>
Phase: <n> · Estimated: <hours>
## Goal
## Context to read
## Tasks
## Acceptance criteria
## Out of scope
## Concepts to explain
```

## Handoff template
```
# Handoff NNN — <task>
Date:
## Done
## Decisions made (and ADRs written)
## Files created or modified
## Known issues / blockers
## What I learned (for interview prep)
## Suggested next step
```
