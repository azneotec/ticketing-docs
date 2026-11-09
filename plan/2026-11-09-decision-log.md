# Decision log — planning sessions (2026-11-06 to 2026-11-09)
Why each decision was made, in the order it was discussed. Use this as raw material for ADRs and interview answers.

## Round 1 — framing
- **Who it's for:** Azad's own learning. Start from basics: assume tools, architecture and infrastructure are unknown or only partly known; aim for full understanding of why each is used.
- **Why distributed:** to experience distributed transactions, sagas, and data consistency across services first-hand.
- **Clean architecture:** strict hexagonal inside each service; distributed architecture between services.
- **Clients:** web only, microfrontends (React + Module Federation). Mobile (Expo/React Native) later, out of scope.
- **Communication:** REST client→gateway/BFF; gRPC internal; Kafka for events; Temporal for workflows.
- **Kubernetes:** introduced once there is more than one service; all depth levels; minikube + Helm.
- **Done means:** a deployed system + a repo that supports talking through trade-offs in interviews.

## Round 2 — domain and sagas
- **Domain:** event ticketing (fresh domain, chosen over order fulfilment, travel booking, and wallet). Seat holds with expiry, contention and idempotency make it naturally distributed.
- **Temporal tension:** Temporal would hide the saga mechanics. Decision: build sagas by hand first, then add Temporal and compare.
- **Saga styles:** experience both choreography and orchestration.
- **Consistency:** transactional outbox plus the other patterns ticketing needs.
- **Stores:** fit for purpose; 24 GB RAM machine (later corrected: 16 GB, see Round 8).
- **Auth:** Keycloak with social login; both gateway and BFF.
- **Observability:** from the beginning.
- **Testing:** Testcontainers, Pact, k6; chaos optional.

## Round 3 — scope vs time
- Initial 4 h/week was ~3–5 years of work for the full scope; raised to **14 h/week** (~12–15 months).
- **MongoDB dropped:** Postgres JSONB covers flexible schema + indexes. Learn **CDC** to keep Elasticsearch in sync.
- **Overselling:** explore both Redis-first and Postgres locking; measure with k6.
- **Flash-sale waiting room:** out of scope now, later learning item.
- **Payment:** fake service under full control (inject failures, delays, duplicate callbacks).
- **Local runtime:** Docker Compose for daily dev, minikube for Kubernetes learning.
- **Java 25 (not Kotlin):** deliberate — strong in Java 8, closing gaps to modern Java.
- **ADRs:** simple, readable, usable for interview prep.

### Auth design (agreed)
Keycloak as IdP with identity brokering for social login. BFF uses Authorization Code + PKCE and keeps tokens server-side (token-handler pattern); browser gets only an httpOnly, SameSite cookie. Gateway does routing, rate limits, early rejection. Every service validates the JWT (zero trust); fine-grained rules live in the domain layer. Service-to-service gRPC forwards the user token or uses client credentials; mTLS later.

### Delivery pipeline (explained)
CI (GitHub Actions) → image registry (GHCR or local) → deploy (manual Helm chosen; Argo CD optional later) → autoscaling (HPA; KEDA on Kafka lag is an option). Cloud CI runners cannot reach local minikube.

## Round 4 — repos and contracts
- **Polyrepo**, structured like an enterprise team, costs accepted (shared contracts repo, platform repo, Pact Broker, multi-PR changes).
- **Schema evolution** with schema registry and compatibility rules.
- **Inventory and Booking separate.**
- **Exactly-once:** try both Kafka transactions and at-least-once + idempotent consumers.
- **Pure DDD domain:** no framework dependencies.
- **Java features:** used deliberately, each with an ADR. Gradle Kotlin DSL.
- **Microfrontends:** shell + Discover, Checkout, Account; built from the early stage.
- **Learning rhythm:** concept → minimal build → break → measure → ADR; explanations come from sessions.

## Round 5 — Inventory/Booking split (interview answer)
Final version: *The separation is driven by failure isolation and different consistency lifecycles: a payment gateway or database outage must not stop users browsing and holding seats. Inventory handles intense write contention on a single invariant through a shared cache (Redis/Valkey) backed by Postgres, independent of the slower booking lifecycle and payment flow.*
Challenges considered and why the first draft (thread exhaustion) was weakened:
- A non-blocking monolith would not hold threads during payment (reserve, commit, handle webhook later).
- Virtual threads (Java 25) make blocked threads cheap.
- "In-memory locks" don't work across pods — say Redis/Postgres precisely.
- Flash sales are deferred, so don't lean on them.
- Keep: independent scaling, failure isolation, different consistency needs. Payment owns money; Booking owns booking lifecycle.

Other round 5 decisions: Spring Boot has no LTS (use latest stable); Gradle Kotlin DSL; trunk-based on master, semver, 80% coverage, shared convention plugin (no Renovate, Sonar, Trivy for now); OpenBao for secrets; PII handling in scope; event cancellation + refunds in scope; SLOs defined; Temporal and hand-built strategies switchable by config; blockers time-boxed.

## Round 6 — secrets, refunds, SLOs
- **Strategy switching:** run both side by side until old sagas finish.
- **Coverage:** 80% per service, same for all modules.
- **Secrets:** External Secrets Operator over agent sidecar — one install, services unaware of OpenBao (fits hexagonal), works with GitOps. Trade-off: secrets live in etcd (base64) — enable etcd encryption as an exercise.
- **Crypto-shredding:** in, via OpenBao transit (per-user key; destroy key to "delete" data, including in old Kafka messages).
- **Refunds:** durable work queue with bounded concurrency (not a waiting room): Postgres refund rows as truth → outbox → Kafka `refund-requested` keyed by booking ID → workers with idempotency key → retry topics + DLT → reconciliation sweep guarantees none lost. Temporal variant: parent workflow + child workflows + continue-as-new.
- **SLOs:** all five confirmed.
- **First milestone:** agreed (Phase 1 above).

## Round 7 — consistency details
- Saga strategy stored on the booking record (truth) **and** propagated in event headers so choreography participants know whether to self-compensate.
- **Redis is a fast front, Postgres the authority:** Redis atomic check-and-set rejects obvious conflicts; Postgres conditional update confirms and wins disagreements.
- **Hold expiry:** `expires_at` column + sweeper in Postgres. On Redis outage, fall back to Postgres-only. Sweeper deletes the Redis key and publishes `SeatReleased`; Redis TTL slightly longer than `expires_at` as safety net; Redis re-warmed from Postgres after an outage.
- **API versioning:** schema registry and Pact rules; Pact also covers frontend→BFF.
- **Backup/DR:** out of scope locally (ADR). OpenBao still runs with persistent storage and a documented unseal procedure.
- **Docs:** ADRs + C4 diagrams + runbooks.
- **Deploys:** manual to minikube (Helm).

## Working model (2026-11-08/09)
- **Elephant and goldfish:** the planning chat (elephant) keeps long memory and writes one brief per session; each Claude Code session (goldfish) starts fresh, reads `CLAUDE.md`, the plan and its brief, builds, and writes a handoff note.
- Mac: Apple Silicon, OrbStack, Claude Code installed.
- Phase 1 shortcut: web shell calls Catalogue directly until the Gateway/BFF arrives.
- Project folder: `~/learn/java_workspace/distributed-ticketing`.

## Round 8 — hardware constraint (2026-11-09)
- Workstation is a **MacBook M1 base, 16 GB RAM** (not 24 GB). macOS + IntelliJ + browser use ~7–8 GB, leaving ~8 GB for containers; the full stack would need 14–18 GB.
- Rule: **run only what the current phase needs.** Never run Compose and minikube at the same time.
- OrbStack memory capped at ~8 GB.
- Observability via the single `grafana/otel-lgtm` image (saves ~1.5 GB vs five containers).
- **Postgres: one shared container, separate database and user per service** (chosen over one container per service, ~1 GB saved). Each service still owns its schema; no cross-database access. Trade-off: shared failure and resource domain locally — recorded in ADR-028.
- Spring services with small heaps and virtual threads; stub services not being worked on.
- Kafka: single KRaft broker, 512 MB heap. Elasticsearch: single node, small heap, only during search work. Temporal: dev server.
- minikube (Phase 4): ~6–7 GB, deploy subsets of services per Helm scenario.
- k6 results are relative comparisons, not capacity claims.
- Fallback: cheap cloud VM for Phase 4+ if it gets too tight (decide then).
- New ADR-027: running a distributed system on constrained hardware.

## Round 9 — after session 001 (2026-11-09)
- Session 001 done: JDK 25.0.4 (Temurin), Gradle 9.9.0, OrbStack 2.2.5 capped at 8 GB, Node 26 LTS via nvm, pnpm via Homebrew (Corepack deprecated), Helm 4, kubectl, minikube, grpcurl.
- Azad removed Homebrew `node` and the Corepack cache.
- Git identity confirmed: Azad Asanali <azneotech@gmail.com>.
- **Docs home (option A):** `ticketing-docs` repo holds the plan, session briefs/handoffs, ADRs, later C4 and runbooks, so everything is versioned. The top-level folder keeps only `AGENTS.md`/`CLAUDE.md`.
- **Agent rules:** `AGENTS.md` is the single source; `CLAUDE.md` contains only `@AGENTS.md`, so Claude Code and Codex share one rule set.
- **GitHub:** repos are pushed to GitHub from session 002 on.
