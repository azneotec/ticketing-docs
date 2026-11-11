# ADR-003: One Git repo per service (polyrepo)

- **Status:** Accepted
- **Date:** 2026-11-09 (decided in planning round 4)
- **Status note:** Partially superseded by ADR-037 for front-end apps.

## Context
The system is made of microservices with a database per service, several frontends and shared contracts. The project should be structured the way an enterprise team works, so that the problems of independent services (versioning, contracts, coordinated changes) are experienced, not avoided.

## Decision
- **Polyrepo:** one Git repo per service and per shared concern, as subfolders of the top-level project folder (see the repo map in the plan).
- Each repo is trunk-based on `master` with its own semantic version.
- Shared build rules live in a Gradle convention plugin repo (`ticketing-build-conventions`).

## Alternatives considered
- **Monorepo** (all services in one repo) — not chosen. It would be faster for a solo developer, but it hides versioning and contract problems and does not mirror enterprise teams that own and release services independently. Polyrepo was chosen for realism; its costs (below) are accepted deliberately as learning and interview material.

## Consequences
**Good**
- Each service builds, versions and releases independently, like in an enterprise team.
- The coordination problems of independent services (contracts, versioning, ordered rollouts) are experienced first-hand, which is the point of the project.

**Bad (costs we accept)**
- A shared **contracts repo** (`ticketing-contracts`) is needed for `.proto` files and event schemas, published as versioned artifacts.
- A **platform repo** (`ticketing-platform`) is needed for Compose, Helm, Kubernetes manifests and observability.
- A **Pact Broker** is needed so consumer and provider contracts can be checked across repos.
- A change that crosses services needs **multiple PRs** in the right order (contract first, then provider, then consumer).

## Interview one-liner
"I used one repo per service to work like an enterprise team; the price is a shared contracts repo, a platform repo, a Pact Broker and multi-PR changes, which is exactly the coordination cost I wanted to learn to manage."
