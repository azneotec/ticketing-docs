# ADR-036: Browser tracing with W3C trace context, exported through a same-origin OTLP route

- **Status:** Accepted
- **Date:** 2026-11-12 (built in session 008; closes the Phase 1 gate)

## Context
Observability from day one (ADR-007) has to start where the user is, in the browser. Otherwise a slow page can't be told apart from a slow service. The Phase 1 gate asks for one trace from the browser through catalogue-service to Postgres. The services already join traces via W3C `traceparent` (ADR-034). The browser must create the first span, send `traceparent` across origins (CORS), and get its spans to the collector.

## Decision
- **OpenTelemetry web SDK, set up once in the host** (`web-shell/src/telemetry.ts`) before anything fetches. Remotes run in the same window, so they are covered without their own setup.
- **Fetch instrumentation** makes a client span per `fetch` and injects **`traceparent`** only for configured API origins (`PUBLIC_TRACE_PROPAGATE_TO`, today `http://localhost:8080`). Other origins never get it.
- **CORS:** `traceparent` makes requests non-simple, so the browser sends a preflight. catalogue-service's CORS for `/events/**` already allows any request header (Spring's default), so **no service change**: the planned `v0.1.1` patch was not needed. Tracing adds one cached preflight (`max-age` 3600) per URL.
- **Export:** a batch span processor sends OTLP/HTTP JSON to the **same-origin path `/otlp/v1/traces`**. The Rsbuild dev server proxies `/otlp` to the collector (`otel-lgtm`, `:4318`). In Phase 2 the gateway takes over the route.
- **Resource:** `service.name=web-shell`, `service.version` from `package.json`, `deployment.environment.name=local`. Always sample locally.
- **Versions:** the OTel JS **2.x** line (`sdk-trace-web` 2.12, instrumentation and exporter 0.223, `@opentelemetry/api` 1.9). The 3.x line (released 2026-10-20) has no web tracer provider yet, and mixing lines breaks the `@opentelemetry/api` peer ranges.

## Alternatives considered
- **Export directly to `:4318` from the browser** — works locally, because `otel-lgtm`'s collector happens to allow any origin. But the browser would need the collector's address, the collector would need public CORS, and each export would cost a preflight. A real collector shouldn't be open to the internet; a same-origin route keeps that decision server-side.
- **Only propagate `traceparent`, no browser SDK** — the server spans would join a trace, but its root (the user's request, including network time) would be missing, and the gate asks for the browser span.
- **Propagate to all origins** — leaks trace ids to third parties, and their servers would reject the preflight, breaking the calls.
- **A vendor RUM product** — vendor lock-in, and it doesn't use our OTLP pipeline.
- **OTel JS 3.x** — newer, but no web provider yet. Revisit when `sdk-trace-web` (or its successor) ships on 3.x.

## Consequences
**Good**
- One trace shows the whole request: browser `GET /events/{id}` → catalogue `http get /events/{id}` → `GetEvent` → JDBC `connection`/`query`. Verified in Tempo in session 008 (trace `ea95a7206dcd44d009c2387437b2aa9c`); this is the Phase 1 gate trace.
- Network time (browser span minus server span) becomes visible.
- New remotes get tracing for free. New APIs only need adding to `PUBLIC_TRACE_PROPAGATE_TO` and allowing `traceparent` in their CORS.

**Bad (costs we accept)**
- An extra preflight per new URL, and about 24 kB gzipped of OTel code in the shell (production build: 188 → 212 kB).
- In dev, React StrictMode mounts twice, so some browser spans are cancelled requests with no server part. That's noise, not a bug.
- `/otlp` exists only on the dev server today. A production build needs the gateway route (Phase 2).
- Browser spans can be faked by any user. Traces are diagnostic data, never an audit trail.
- On the older OTel line until 3.x supports the web.

## Interview one-liner
"The shell starts the OpenTelemetry web SDK once, so every fetch, including the remotes', is a span that sends a W3C `traceparent` to our own APIs only. The service continues the trace down to SQL, so one trace runs from the click to the query. Spans leave through a same-origin `/otlp` route instead of exposing the collector, and adding the header cost exactly one CORS preflight."
