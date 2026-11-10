# ADR-007: Observability from day one (OpenTelemetry, `grafana/otel-lgtm` locally)

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning rounds 2 and 8; local stack built in session 005)

## Context
A distributed system fails between services: a slow booking may be caused by Inventory, Kafka lag or Postgres. Without traces, metrics and logs that share one trace ID, those failures are guesswork. Adding telemetry late means retrofitting every service, so the plan makes observability a day-one rule, and the Phase 1 gate is a request traced browser → Catalogue → Postgres in Grafana/Tempo. The laptop has 16 GB RAM (ADR-027): running Grafana, Loki, Tempo, Prometheus and an OTel Collector as five containers costs about 1.5 GB more than one combined image (decision log, round 8).

## Decision
- Every service emits traces, metrics and logs with **OpenTelemetry** and exports them over **OTLP** (gRPC 4317 / HTTP 4318). Services know only the OTLP endpoint, never the backend.
- Locally, the backend is the single **`grafana/otel-lgtm`** container (OTel Collector + Loki + Grafana + Tempo + Prometheus + Pyroscope) in the `ticketing-platform` Compose `core` profile, pinned (`0.40.0`), capped at 1.5 GB, data in a named volume.
- The platform smoke test sends a real OTLP span and checks Tempo stored it, so the pipeline is proven before any service exists.
- **In Kubernetes (Phase 4)** the parts run separately (Helm charts for Grafana, Loki, Tempo, Prometheus, and an OTel Collector), with retention and resource limits per part. Services do not change: only the `OTEL_EXPORTER_OTLP_ENDPOINT` they are given.

## Alternatives considered
- **Add observability later** — cheaper at first, but every service would be retrofitted, and the Phase 2 saga debugging would be blind.
- **Five separate containers locally** — closer to production, but ~1.5 GB more memory than the budget allows.
- **Vendor agents or a SaaS backend (Datadog, New Relic, Grafana Cloud)** — little to run, but sends data off the laptop, adds accounts and cost, and hides how the pieces work, which is the learning goal.
- **Zipkin/Jaeger + ELK** — mature, but more containers and more memory; OTel + LGTM covers all three signals with one protocol.

## Consequences
**Good**
- One trace ID ties a request across services, Kafka and Postgres; logs link to traces in Grafana.
- Backend-neutral: OTLP lets us swap the local all-in-one for separate components (or a vendor) without touching services.
- Small: about 0.5–0.7 GB measured at idle.

**Bad (costs we accept)**
- The all-in-one image is a development tool: no high availability, short retention, one failure domain, and its versions move together.
- Local Grafana runs with anonymous admin access (fine on 127.0.0.1, never elsewhere).
- Telemetry in every service costs some CPU, memory and setup work from the first service on.

## Interview one-liner
"Every service speaks OpenTelemetry over OTLP from the first commit, because distributed failures happen between services; locally one `otel-lgtm` container gives traces, metrics and logs in Grafana for about half a gigabyte, and in Kubernetes the same OTLP stream goes to separate Tempo, Loki and Prometheus without changing the services."
