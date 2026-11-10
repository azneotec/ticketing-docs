# ADR-027: Running a distributed system on constrained hardware (16 GB laptop)

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning round 8; first applied in session 005)

## Context
The workstation is a MacBook M1 with **16 GB RAM**, not the 24 GB first assumed. macOS, IntelliJ and a browser use about 7–8 GB, leaving roughly 8 GB for containers. The full target stack (about ten Spring services, Postgres, Redis, Kafka, Elasticsearch, Keycloak, Temporal, OpenBao, observability, a schema registry, then minikube) would need 14–18 GB. Without rules, the machine swaps, containers get OOM-killed and measurements become meaningless.

## Decision
- **Run only what the current task needs.** Infrastructure is grouped into Compose **profiles** (`core` = Postgres + otel-lgtm, `messaging` = Kafka, `tools` = Kafbat UI; later Redis, Apicurio, Keycloak, Elasticsearch get their own). Services not being worked on are stubbed or stopped.
- **Every container has a memory limit** (`mem_limit`), and every JVM's heap sits well below it (Kafka 512 MB heap in 1 GB; Spring services `-Xmx256m`–`384m` with virtual threads).
- **Shrink the shared parts:** one Postgres container for all services (ADR-028); one `grafana/otel-lgtm` container (ADR-007); a single KRaft Kafka broker; Elasticsearch single node, small heap, only during search work; Temporal as `temporal server start-dev`.
- **OrbStack is capped at ~8 GB.** **Never run Compose and minikube at the same time.** minikube (Phase 4) gets ~6–7 GB and runs subsets of services per Helm scenario.
- Pinned images, ports on `127.0.0.1`, and `make status` showing memory per container and the total, so the budget is visible.
- k6 results are read as **relative comparisons**, never capacity claims. If Phase 4+ does not fit, fall back to a cheap cloud VM (decide then).

## Alternatives considered
- **Buy more RAM / a new machine** — removes the problem, but not available now.
- **A cloud VM or Kubernetes cluster from the start** — enough memory, but costs money every month, adds network latency to every edit-run cycle, and needs remote setup before any learning starts. Kept as a fallback.
- **Run everything and let the OS swap** — simplest config, but slow, unpredictable and makes performance experiments worthless.

## Consequences
**Good**
- Measured on 2026-11-10: `core` + `messaging` uses about **0.9 GB** at idle (limits total 3 GB); with `tools` about 1.4 GB. Plenty of room for the first services.
- Hard limits make memory problems visible early (OOM kill, exit 137) instead of slowing the whole laptop.
- Learning value: sizing heaps vs container limits is a real production skill.

**Bad (costs we accept)**
- Never the full system at once; end-to-end runs need deliberate subsets.
- Single-node Kafka and shared Postgres hide replication and failover behaviour; those are learned in minikube or not at all locally.
- Performance numbers are only comparable with each other, not with production.
- Switching between Compose and minikube costs time.

## Interview one-liner
"I built a ten-service system on a 16 GB laptop by treating memory as a budget: profiles so only the current slice runs, hard limits on every container with JVM heaps sized under them, shared Postgres and an all-in-one observability image, and never Compose and Kubernetes together — accepting that I never see the whole system at once and that load-test numbers are only relative."
