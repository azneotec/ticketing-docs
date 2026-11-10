# Handoff 005 — Version fix + `ticketing-platform`
Date: 2026-11-10

## Done
Completed in one session (no 005a/005b split).

**Part A — fixes**
- `ticketing-docs`: decision log **Round 12** records:
  - Buf stays on FILE rules
  - ADR-010's reasons confirmed
  - Kafbat UI in the `tools` profile
  - shared Postgres and otel-lgtm
  - the versioning fix

  Brief 005 is committed. **ADR-010 is unchanged**: it already gives the confirmed reasons (browser/cookie/BFF fit, typed generated clients, binary HTTP/2 calls, deadlines).
- `gradle/version.gradle.kts` (conventions and contracts, still the same file in both): if `git status --porcelain` prints anything, the tree is dirty, and the version is the **next `-SNAPSHOT`**, even on a release tag. A clean tree on `vX.Y.Z` still gives `X.Y.Z`. If git can't run, the tree counts as dirty.
- New cases in both failure scripts: copy the repo, tag `v9.9.9`, expect `9.9.9`, add a file, expect `9.9.10-SNAPSHOT`. **Conventions: 8 passed, 0 failed. Contracts: 8 passed, 0 failed.**
- Conventions README "Releasing" step 6: after moving the sample to the next `-SNAPSHOT`, commit and run `./gradlew publishToMavenLocal`. Both READMEs now describe the dirty-tree rule; the old warning note in the contracts README is gone.
- **Releases (pushed, tags on GitHub):**
  - conventions **v0.1.2**. The sample now points at `0.1.3-SNAPSHOT`, which is published to mavenLocal.
  - contracts **v0.1.1**, now built with conventions 0.1.2 (Azad's choice).

  Both are in `~/.m2`. `git push` and tag pushes worked from Claude Code; only `gh repo create` is blocked.

**Part B — `ticketing-platform` (new repo, committed locally, not on GitHub yet)**
- `compose.yaml` (project `ticketing`, network `ticketing`, named volumes, every port bound to `127.0.0.1`, health checks, `mem_limit` everywhere):

  | Service | Image | Profile | Limit | Notes |
  |---|---|---|---|---|
  | postgres | `postgres:18.6-alpine` | core | 512 MB | `shared_buffers=64MB`, `work_mem=4MB`, `max_connections=60`, `wal_level=logical` (for Phase 3 CDC) |
  | otel-lgtm | `grafana/otel-lgtm:0.40.0` | core | 1.5 GB | Grafana 3000, OTLP 4317/4318, data in `/data` volume, image's own health check |
  | kafka | `apache/kafka:4.4.0` | messaging | 1 GB | KRaft combined node, heap 512 MB, listeners INTERNAL `kafka:19092`, EXTERNAL `localhost:9092`, CONTROLLER 9093 |
  | kafbat-ui | `kafbat/kafka-ui:v1.5.0` | tools | 384 MB | http://localhost:8089, heap 192 MB |

- `postgres/initdb/01-service-databases.sh`:
  - one login + database per service (`catalogue`, `inventory`, `booking`, `payment`)
  - each login owns its database
  - `REVOKE CONNECT, TEMPORARY … FROM PUBLIC` on each of them and on the admin `postgres` database
  - passwords passed as psql variables (safely quoted)
- `.env.example` (committed) and `.env` (ignored, created by `make up`). The README explains that OpenBao replaces it in Phase 4.
- `Makefile`:
  - `up`: core + messaging, `--wait` for healthy
  - `tools`, `down`, `logs`
  - `status`: `ps` + `docker stats --no-stream` per container + TOTAL
  - `smoke`
  - `reset`: you must type `yes`; tested that anything else aborts
- `scripts/smoke-test.sh`. Result: **12 passed, 0 failed**, on a running stack and again from empty volumes after `reset`:
  - each login connects to its own database and creates a table there, and is **refused on the next service's database** (`permission denied for database "inventory"` etc.)
  - topic `smoke-test`: one unique message produced and consumed via `kafka:19092`, then read again via `localhost:9092` from a host-networked container. That proves both advertised listeners.
  - one span sent with curl to OTLP/HTTP 4318, then found in Tempo through Grafana's data-source proxy. The script prints the trace ID and a direct Grafana Explore link.
- **Measured memory:**
  - core + messaging: **~0.9–1.0 GB** at idle (limits total 3 GB)
  - with `tools`: ~1.4 GB
  - startup to all healthy: ~20 s (~55 s from empty volumes)
- Kafbat UI (`make tools`) is healthy and lists `smoke-test` (checked via its API). Topics appear a few seconds after it starts.
- Trace check: I opened the printed link in a browser. Grafana showed trace `smoke-test / smoke-test-span`, 250 ms. **Azad still has to confirm it by eye**: run `make smoke` and open the link.
- README covers quick start, ports, how to connect from the host and from containers, secrets, the smoke test, and the brief's concepts.
- ADR-007, ADR-027 and ADR-028 are written and indexed.

## Decisions made (and ADRs written)
- **ADR-007** observability from day one. **ADR-027** constrained hardware. **ADR-028** shared Postgres, one database per service. All indexed.
- **Image pins:** the newest non-RC release of each image on 2026-11-10. Postgres uses the Alpine variant (smaller image). With Postgres 18 the volume is mounted at `/var/lib/postgresql` (the new data layout).
- **Kafka health check:** `kafka-cluster.sh api-versions` with `KAFKA_HEAP_OPTS=-Xmx64m`, so the check's own JVM fits in the 1 GB container next to the broker. `kafka-broker-api-versions.sh` is deprecated in 4.4.
- **Kafka CLI in the smoke test** runs in throw-away containers (`--memory 384m`, heap 128 MB), never inside the broker container, so the CLI tools never compete with the broker's memory.
- **Fixed `CLUSTER_ID`** in `compose.yaml`, so the Kafka volume survives restarts. Not a secret.
- **Kafbat on host port 8089**, keeping 8080 free for Spring services.
- **All ports bound to `127.0.0.1`**: nothing is reachable from the LAN. Local Grafana runs as anonymous admin, which is only acceptable because of that.
- **`wal_level=logical`** is set now, so Phase 3 Debezium needs no Postgres restart or reconfiguration.
- **Dirty-tree check fails safe:** if git isn't available, the version is treated as dirty and gets `-SNAPSHOT`.

## Files created or modified
`ticketing-docs/` (pushed):
- `plan/2026-11-09-decision-log.md` (Round 12)
- `sessions/005-ticketing-platform.md` (brief, committed), `sessions/005-ticketing-platform-handoff.md` (this file)
- `decisions/ADR-007-observability-day-one.md`, `decisions/ADR-027-constrained-hardware.md`, `decisions/ADR-028-shared-postgres-database-per-service.md` (new), `decisions/README.md` (index)

`ticketing-build-conventions/` (pushed, tag `v0.1.2`):
- `gradle/version.gradle.kts`
- `samples/verify-failure-cases.sh` (case 6, version)
- `README.md` (version table, release step 6)
- `samples/hexagonal-sample/gradle.properties` (0.1.2, then 0.1.3-SNAPSHOT)

`ticketing-contracts/` (pushed, tag `v0.1.1`):
- `gradle/version.gradle.kts`
- `samples/verify-failure-cases.sh` (case 6, version)
- `README.md` (versioning paragraph)
- `gradle.properties` (conventions 0.1.2)

`ticketing-platform/` (new repo, one commit on `master`, **not pushed yet**):
- `.editorconfig`, `.gitignore`, `.env.example`, `README.md`, `Makefile`, `compose.yaml`
- `postgres/initdb/01-service-databases.sh`, `scripts/smoke-test.sh`
- `.env` (local, git-ignored)

Outside the project:
- Docker images pulled: `postgres:18.6-alpine`, `apache/kafka:4.4.0`, `grafana/otel-lgtm:0.40.0`, `kafbat/kafka-ui:v1.5.0`
- Docker volumes `ticketing_postgres-data`, `ticketing_kafka-data`, `ticketing_otel-lgtm-data` and network `ticketing`
- `~/.m2/repository/io/github/azneotec/ticketing/`: conventions `0.1.2` and `0.1.2-SNAPSHOT` / `0.1.3-SNAPSHOT`, and `ticketing-contracts-{grpc,avro}/0.1.1`

## Known issues / blockers
1. **`ticketing-platform` isn't on GitHub yet.** Azad runs, from `ticketing-platform/`:
   `gh repo create azneotec/ticketing-platform --public --source . --push`
2. **Trace by eye** (acceptance): Azad runs `make smoke` and opens the printed Grafana link. The stack was left running after this session; `make down` stops it.
3. `otel-lgtm` also runs Pyroscope (profiles), which the brief didn't ask for. The image can't turn it off. It fits in the limit, but the container's idle memory will grow with stored data. Watch `make status`.
4. `make up` uses `--wait`, so if a container never becomes healthy, it fails after the health-check retries. Use `make logs s=<service>`.
5. Kafbat's topic list is empty for a few seconds after start, while it loads cluster metadata.
6. Javadoc "no @return" warnings in conventions `arch-rules` (from before this session; they don't fail the build).
7. Still open from handoff 004: mavenLocal is the only repository; Buf must be installed on every machine.

## What I learned (for interview prep)
- **Compose networks, volumes, health checks, profiles.**
  - A network is a private LAN with DNS by service name.
  - A named volume outlives containers (`down` keeps it, `down -v` deletes it).
  - A health check is a command Docker repeats; `up --wait` and `depends_on: service_healthy` rely on it.
  - Profiles group services so only what's needed starts.
- **Memory limit vs heap.** `mem_limit` is a kernel cgroup cap; going over it means an OOM kill (exit 137). `-Xmx` covers only the heap. Metaspace, thread stacks, direct buffers and the JIT come on top, so the heap must stay well under the limit. Kafka: 512 MB heap in a 1 GB container, measured at ~400 MB.
- **KRaft vs ZooKeeper.** Kafka metadata used to live in a separate ZooKeeper cluster. KRaft keeps it inside Kafka: controller nodes agree on it with Raft. Kafka 4 removed ZooKeeper. A single node can be both controller and broker.
- **Listeners vs advertised listeners.** A listener is a socket the broker opens. An advertised listener is the address the broker *tells* clients to reconnect to after bootstrap. It must be reachable from where the client runs, hence `kafka:19092` for containers and `localhost:9092` for the host. A wrong advertised address is the classic "connects, then hangs" bug.
- **Postgres roles, databases, `CONNECT`.** `CONNECT` on a database is granted to `PUBLIC` by default. Revoking it and granting it only to the owner turns "a database per service" from a convention into an enforced rule.
- **OTLP.** The OpenTelemetry Protocol, over gRPC (4317) or HTTP (4318, protobuf or JSON). Services export OTLP and don't know the backend. A trace is just JSON: trace ID (16 bytes), span ID (8 bytes) and timestamps in nanoseconds.
- **LGTM.** The parts:
  - Loki: logs
  - Grafana: UI
  - Tempo: traces
  - Mimir: metrics, with Prometheus standing in for it locally

  The OTel Collector in front receives OTLP and routes each signal to its store. Here they all run in one container; in Kubernetes each part runs separately.
- **Release hygiene.** A version number must identify exactly one set of contents. Building a release version from uncommitted work breaks that, so a dirty tree is always a snapshot.

## Suggested next step
Phase 1 step 6: `catalogue-service`. It will consume:
- conventions 0.1.2
- contracts 0.1.1
- the `catalogue` database (`localhost:5432`, login `catalogue`)
- OTLP to `localhost:4317/4318`

Work items: hexagonal modules + ArchUnit, Flyway, JSONB seat maps, Testcontainers, OTel, and ADR-004..006 (ADR-004 already exists; 005 Java 25 features, 006 JSONB over MongoDB). The service only needs `make up PROFILES="core"` (Postgres + otel-lgtm, ~0.6 GB) until it publishes events.
