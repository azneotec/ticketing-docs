# Session 005 — Version fix + `ticketing-platform` (local Compose platform)
Phase: 1 · Estimated: 4–5 hours (split into 005a / 005b if needed; handoff at the split)

## Goal
Fix the release-version trap, then build a local platform repo that starts Postgres, Kafka and the observability stack with one command, inside the 16 GB memory budget.

## Context to read
- `AGENTS.md`
- `ticketing-docs/plan/2026-11-08-project-plan.md` (Hardware, Resource budget, Stores rows)
- `ticketing-docs/plan/2026-11-09-decision-log.md` (Round 8 hardware)
- `ticketing-docs/sessions/004-ticketing-contracts-handoff.md` (known issues 1–2)

## Decisions already made (record them; do not re-decide)
- Buf stays on **FILE** breaking rules (in a polyrepo, consumers compile against generated Java, so source-level breaks matter, not only wire breaks; loosening later is easy, tightening is hard).
- ADR-010 reasons confirmed: REST fits browsers, cookies and the BFF; gRPC gives typed generated clients, efficient binary calls and deadlines between services.
- **Kafbat UI** for browsing Kafka, under an optional `tools` profile.
- One shared Postgres container, one database + login per service (ADR-028). Observability via `grafana/otel-lgtm`.

## Part A — fixes
1. `ticketing-docs`: append **Round 12** to the decision log (the decisions above). Update ADR-010 only if its reasons differ from the confirmed ones.
2. Versioning (`gradle/version.gradle.kts`) in `ticketing-build-conventions` and `ticketing-contracts`: if the working tree is dirty (`git status --porcelain` non-empty), the version is always `-SNAPSHOT`, even on a release tag. Add a test case to each failure script proving it.
3. Conventions README "Releasing": add the `publishToMavenLocal` step for the new snapshot.
4. Release conventions **0.1.2** and contracts **0.1.1** following each README. Give Azad the exact tag-push commands if blocked.

## Part B — `ticketing-platform` repo
5. `git init` on `master`, `.editorconfig`, `.gitignore` (includes `.env`), README.
6. `compose.yaml` with pinned image versions (no `latest`), a named network, named volumes, health checks and **memory limits on every service**:
   - **postgres** (18.x): `initdb` script creating databases + logins for `catalogue`, `inventory`, `booking`, `payment` (each login can connect only to its own database; revoke `CONNECT` from `PUBLIC`). Tuned small (`shared_buffers` etc.). Limit ~512 MB.
   - **kafka** (`apache/kafka`, single broker, KRaft, combined controller+broker), heap 512 MB, limit ~1 GB; advertised listeners working both from the host and from other containers (explain the two listeners).
   - **otel-lgtm** (`grafana/otel-lgtm`): Grafana UI, OTLP gRPC 4317 and HTTP 4318. Limit ~1.5 GB.
   - **kafbat-ui** in profile `tools`, limit ~384 MB.
   - Profiles: `core` = postgres + otel-lgtm; `messaging` = kafka; `tools` = kafbat-ui. Target: `core` + `messaging` under ~3 GB.
7. Secrets for local only: `.env.example` committed, `.env` ignored; README explains OpenBao replaces this in Phase 4.
8. `Makefile` (or a small script) with `up`, `down`, `status` (shows `docker stats --no-stream` memory per container), `logs`, `reset` (destroys volumes — must ask for confirmation).
9. `scripts/smoke-test.sh`: each service login can connect to its own DB and is **refused** on another's; create a topic, produce and consume one message; send one test trace via OTLP (e.g. `otel-cli` or a curl to 4318) and show where to find it in Grafana Tempo. Print a clear pass/fail summary.
10. ADRs in `ticketing-docs/decisions/` (update index):
    - **ADR-007** observability from day one (otel-lgtm locally; what changes in Kubernetes).
    - **ADR-027** running on constrained hardware (16 GB, profiles, limits, never Compose and minikube together).
    - **ADR-028** shared Postgres container, database per service (isolation by login, trade-offs).
11. Conventional Commits. **Azad runs** `gh repo create azneotec/ticketing-platform --public --source . --push`.

## Acceptance criteria
- Fixes released: conventions `v0.1.2`, contracts `v0.1.1` on GitHub; dirty-tree builds are always `-SNAPSHOT`.
- `make up` (core + messaging) is healthy; `make status` shows total under ~3 GB.
- Smoke test passes, including the cross-database refusal.
- A test trace is visible in Grafana Tempo (Azad confirms by eye).
- Kafbat UI starts with the `tools` profile and shows the smoke-test topic.
- ADR-007, 027, 028 written and indexed; Round 12 recorded; all repos clean and pushed.

## Out of scope
- Redis, Apicurio, Keycloak, Elasticsearch (later profiles).
- Any Spring service; minikube; OpenBao.

## Concepts to explain (2–4 lines each)
Compose networks, volumes, health checks and profiles · container memory limit vs JVM heap · KRaft vs ZooKeeper · Kafka listeners and advertised listeners · Postgres roles, databases and `CONNECT` privilege · OTLP · what each LGTM part does (Loki, Grafana, Tempo, Mimir/Prometheus) and the OTel Collector inside it.
