# Distributed Ticketing Platform — Project Plan
As of 2026-11-08 · Source of truth: this file (mirrors the Claude doc "2026-11-08-distributed-ticketing-project-plan"). Reasoning: `2026-11-09-decision-log.md`.

## Purpose
Event ticketing platform built as a distributed system, learning each tool from the ground up, finished to senior-engineer standard. Outcome: deployed system on local Kubernetes + repo of ADRs for interview prep. 14 h/week, ~12–15 months.
Learning rhythm per topic: concept → minimal build → deliberate break → measure → ADR.

## Decisions
| Area | Decision |
|---|---|
| Domain | Event ticketing |
| Language/build | Java 25, latest stable Spring Boot, Gradle Kotlin DSL; new Java features used deliberately with ADRs |
| Service design | Strict hexagonal; framework-free domain module; ArchUnit |
| Style | Microservices, database per service |
| Client API | REST via Spring Cloud Gateway + BFF |
| Internal sync | gRPC (never client-facing) |
| Async | Kafka + transactional outbox |
| Exactly-once | Try both: Kafka transactions and at-least-once + idempotent consumers |
| Sagas | Hand-built choreography and orchestration first, then Temporal; strategy selectable by config |
| Strategy switching | Side by side until old sagas finish; strategy stored on booking record (truth), propagated in event headers |
| Stores | PostgreSQL (JSONB instead of MongoDB), Redis/Valkey, Elasticsearch. One shared Postgres container, separate database + user per service (ADR-028) |
| Seat holds | Postgres authority, Redis fast front; `expires_at` + sweeper; sweeper clears Redis; Redis TTL slightly longer as safety net; Postgres fallback if Redis down; re-warm Redis after outage |
| Search | CDC (Debezium) Postgres → Elasticsearch |
| Schemas | Schema registry + compatibility rules; Pact for gRPC, BFF and frontend→BFF |
| Payment | Fake service with switchable failures |
| Refunds | Durable work queue: refund rows in Postgres = truth, outbox → Kafka, bounded concurrency, idempotency key = booking ID, retry topics + DLT, reconciliation sweep; Temporal variant |
| Domain scope | Purchase, organiser event cancellation, refunds. Flash-sale waiting room later |
| Frontend | React + Module Federation: shell + Discover, Checkout, Account; built early. Mobile out of scope |
| Auth | Keycloak + social login; BFF token handler (httpOnly cookie); gateway coarse checks; services validate JWT |
| Secrets | OpenBao (persistent, documented unseal) + External Secrets Operator |
| PII | Log masking, retention, crypto-shredding via OpenBao transit |
| Observability | Day one: OpenTelemetry, Prometheus, Grafana, Loki, Tempo — locally as the single `grafana/otel-lgtm` image |
| Testing | Testcontainers, Pact, k6; 80% coverage per service; chaos optional |
| Infra | Docker Compose (OrbStack) for dev; minikube + Helm, manual deploys, HPA |
| Repos | One per service; trunk-based `master`; semver; shared Gradle convention plugin |
| Docs | ADRs, C4 diagrams, runbooks. `ticketing-docs` repo is home for plan, session briefs/handoffs and ADRs (from session 002) |
| Out of scope | Local backups/DR; waiting room (later); mobile |
| Blockers | Time-box ~3 h, record, move on |
| Phase 1 shortcut | Web shell calls Catalogue REST directly until Gateway/BFF arrives in Phase 2 |
| Hardware | MacBook M1, 16 GB RAM. OrbStack capped at ~8 GB. Run only what the current phase needs; never Compose and minikube together (ADR-027) |
| Resource budget | Spring services `-Xmx256m`–`384m` + virtual threads; stub services not under work; Kafka 1 KRaft broker, 512 MB heap; Elasticsearch single node 512 MB–1 GB heap, only during search work; Temporal via `temporal server start-dev`; minikube ~6–7 GB with a subset of services per Helm scenario; k6 results read as relative comparisons. Fallback: cheap cloud VM for Phase 4+ if needed |
| Agent rules | `AGENTS.md` is the single rules file; `CLAUDE.md` only imports it (`@AGENTS.md`) |
| Remote | Repos pushed to GitHub from session 002 |
| Working model | Elephant (planning chat) writes briefs; goldfish (fresh Claude Code session) builds one brief and writes a handoff |

## Repo map
| Repo | Owns | Phase |
|---|---|---|
| ticketing-platform | Compose, Helm, k8s manifests, secrets config, observability, k6 | 1 |
| ticketing-contracts | .proto and event schemas, versioned artifacts | 1 |
| ticketing-build-conventions | Gradle convention plugin | 1 |
| ticketing-docs | ADRs, C4, runbooks, SLOs | 1 |
| catalogue-service | Events, venues, seat maps (Postgres + JSONB) | 1 |
| web-shell | React host | 1 |
| web-discover | Browse/search remote | 1–3 |
| inventory-service | Seats, holds, no-overselling (Postgres + Redis) | 2 |
| booking-service | Booking lifecycle, orchestrator, refunds | 2 |
| payment-service | Fake payments | 2 |
| gateway-bff | Gateway + BFF | 2 |
| web-checkout | Checkout remote | 2 |
| search-service | Elasticsearch read model via CDC | 3 |
| web-account | Account remote | 4 |
| notification-service | Notifications | 4 |

## Roadmap
1. **Foundations (wk 1–6):** workstation; docs/conventions/contracts/platform repos; Compose with Postgres, Kafka (KRaft), observability; Catalogue service hexagonal + ArchUnit + Flyway + Testcontainers + OTel; React shell listing events; ADR 001–007. **Gate:** request traced browser → Catalogue → Postgres in Grafana/Tempo.
2. **First saga (wk 7–18):** Inventory, Booking, Payment; gRPC; outbox; Gateway/BFF + Keycloak; choreography then hand-built orchestration; Checkout remote; Kafka txns vs idempotency. **Gate:** booking survives a killed pod mid-saga.
3. **Contracts + search (wk 19–26):** schema registry, Pact, CDC → Elasticsearch, Discover remote. **Gate:** search freshness p95 < 5 s.
4. **Kubernetes + ops (wk 27–38):** minikube + Helm, HPA, OpenBao + ESO, cancellation + refund queue, Notification, Account remote, k6 hold comparison. **Gate:** all SLOs green under load.
5. **Advanced (wk 39–52+):** Temporal, side-by-side switching, crypto-shredding, optional chaos; later waiting room, mobile. **Gate:** Temporal vs hand-built ADR written.

### Phase 1 step plan
1. Workstation setup (session 001)
2. `ticketing-docs`: ADR template, ADR-001..003
3. `ticketing-build-conventions`: Java 25 toolchain, JaCoCo 80%, ArchUnit, versioning; publish to mavenLocal
4. `ticketing-contracts`: protobuf build + first Catalogue .proto, versioned artifact
5. `ticketing-platform`: Compose with shared Postgres, Kafka (KRaft, 512 MB heap), `grafana/otel-lgtm`; resource limits per container; ADR-007, ADR-027, ADR-028
6. `catalogue-service`: hexagonal modules, ArchUnit, Flyway, JSONB seat maps, Testcontainers, OTel; ADR-004..006
7. `web-shell`: React host with Module Federation, lists events (direct REST)
8. Gate check + retro

## SLOs
| SLO | Target |
|---|---|
| Seat hold | 99.5% success; p99 < 300 ms |
| No overselling | Zero, checked by reconciliation job |
| Saga completion | 99% final within 15 min; alert if stuck > 30 min |
| Search freshness | p95 < 5 s |
| Refunds after cancellation | 99% within 24 h; zero lost |

## ADR backlog
001 ADR format/location · 002 Ticketing domain · 003 Repo per service · 004 Strict hexagonal + ArchUnit · 005 Java 25 features · 006 JSONB over MongoDB · 007 Observability day one · 008 Inventory/Booking split · 009 Seat-hold design · 010 REST edge, gRPC inside · 011 Gateway + BFF auth · 012 Outbox · 013 Choreography · 014 Hand-built orchestration · 015 Kafka txns vs idempotency · 016 Schema registry · 017 Pact · 018 CDC to Elasticsearch · 019 Module Federation split · 020 Helm/minikube/HPA · 021 OpenBao + ESO · 022 Refund work queue · 023 Temporal vs hand-built · 024 Strategy switching · 025 PII + crypto-shredding · 026 Backup/DR out of scope · 027 Running on constrained hardware (16 GB) · 028 Shared Postgres container, database per service · 029 Formatting enforced by the build (Spotless + google-java-format) · 030 Avro for events, Protobuf for gRPC · 031 Spring Data JPA, entities separate from the domain · 032 REST errors as Problem Details, optimistic locking via ETag/If-Match · 033 Localhost-only admin API until Phase 2 · 034 gRPC server on Spring Boot built-in support, OpenTelemetry via Micrometer (no agent) · 035 Front-end tooling (Rsbuild, Tailwind, TanStack Query, generated OpenAPI types, Vitest/MSW/Playwright) · 036 Browser tracing (W3C trace context end to end, OTLP export path)
