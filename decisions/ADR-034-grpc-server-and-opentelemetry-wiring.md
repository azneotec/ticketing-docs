# ADR-034: gRPC server on Spring Boot's built-in support; OpenTelemetry through Micrometer, no agent

- **Status:** Accepted
- **Date:** 2026-11-11 (options set in planning round 14; checked and decided in session 007)

## Context
ADR-010 puts gRPC between services, and ADR-007 requires traces, metrics and logs over OTLP from day one. `catalogue-service` is the first service to do both, so the choices here become the template for Inventory, Booking and Payment. Round 14 asked for **Spring gRPC if it supports Spring Boot 4.1**, otherwise plain grpc-java run by a Spring lifecycle bean. It also asked for **Spring Boot's built-in OpenTelemetry support, no Java agent** (memory, ADR-027). This ADR records what was checked and how it is wired. It is separate from ADR-007, which states the principle; this one covers the Spring wiring.

## Decision
**gRPC server**
- Spring Boot 4.1 ships Spring gRPC itself: `spring-boot-starter-grpc-server` (Spring gRPC 1.1.2 in the Boot BOM). We use it, so plain grpc-java with a hand-written lifecycle bean isn't needed.
- What we get from it:
  - Any bean of a gRPC service type is served.
  - Port `spring.grpc.server.port` (9090).
  - Server reflection (for `grpcurl`), the standard health service, graceful shutdown.
  - An observation interceptor that gives spans and metrics per RPC.
- The contracts (`ticketing-contracts-grpc` 0.1.1) were built with grpc-java 1.85.1, but the Boot BOM pins 1.84.1. We import `io.grpc:grpc-bom:1.85.1` so every `io.grpc` module resolves to one version: mixed grpc-java versions fail at runtime.
- Status mapping in the adapter:
  - `NotFoundException` → `NOT_FOUND`
  - invalid input → `INVALID_ARGUMENT`
  - illegal transition → `FAILED_PRECONDITION`
  - version conflict → `ABORTED`
  - anything else → `INTERNAL`, with no message sent to the client
- Deadlines are honoured: an already-expired call does no work, and a reply to a cancelled call is not sent.

**OpenTelemetry**
- **Instrument with Micrometer Observation, export with the OpenTelemetry SDK:** `spring-boot-starter-opentelemetry` (Micrometer tracing bridge to OTel, OTLP exporters for traces, metrics and logs). Endpoint `OTEL_EXPORTER_OTLP_ENDPOINT`, default `http://localhost:4318` (OTLP/HTTP).
- Span sources:
  - HTTP and gRPC: Spring Boot.
  - JDBC: `datasource-micrometer` 2.3.0, which wraps the `DataSource` (`connection` and `query` spans).
  - Use cases: an observation per call, added in `bootstrap` by a JDK dynamic proxy around each inbound port. The application module stays framework-free (ADR-004), and each use case also gets a `catalogue.use-case` timer (rate, errors, duration).
- Logs:
  - The console pattern carries `[traceId-spanId]` (Spring Boot's correlation).
  - The OpenTelemetry Logback appender sends the same records to Loki. It is connected to Boot's SDK at startup.
  - The appender version (2.28.0-alpha) is pinned to the instrumentation release built on the OTel version Boot manages (1.62), so API and SDK versions don't mix.
- Resource attributes: `service.name` (from `spring.application.name`), `service.version` (the Gradle version, substituted at build time), `deployment.environment.name`. Sampling is 100% locally (`TRACING_SAMPLING_PROBABILITY`).
- Tests don't export. `TracingTest` uses an in-memory span exporter to prove one request is one trace: HTTP → use case → SQL.

## Alternatives considered
- **Plain grpc-java with our own lifecycle bean** — the fallback round 14 allowed. We'd have to write server start/stop, graceful shutdown, reflection and health registration, and observation interceptors ourselves, which Boot now provides and maintains.
- **`grpc-spring-boot-starter` (net.devh / grpc-ecosystem)** — popular in the Boot 2/3 era, but third-party and lagging Spring Boot 4. Spring's own support makes it unnecessary.
- **OpenTelemetry Java agent** — instruments everything with no code, but costs roughly 50–100 MB more per JVM and slower start-up, and hides how instrumentation works. Ruled out in round 14 for memory (ADR-027).
- **OpenTelemetry Spring Boot starter (from the OTel project)** — a second instrumentation stack next to Micrometer, which Spring already uses everywhere; two ways to make spans in one app.
- **`@Observed` on use cases** — needs AspectJ and a Micrometer annotation inside the application module, which would break ADR-004.

## Consequences
**Good**
- Both choices are Spring-native and documented. Upgrading Spring Boot upgrades gRPC server support and telemetry together.
- One trace shows the whole request inside the service. In Phase 2 the same `traceparent` propagation links Booking → Catalogue over gRPC.
- No agent: measured RSS ~95 MB idle, ~280 MB after traffic with `-Xmx256m` (heap + Netty + OTel SDK).
- The use-case proxy gives RED metrics per use case for free, and it's reusable in every service.

**Bad (costs we accept)**
- Three version pins to keep in step with Spring Boot: the grpc BOM (to the contracts' grpc-java), `datasource-micrometer`, and the alpha-versioned Logback appender. Check them on every Boot upgrade.
- Logs reach Loki only after Spring has started (the appender is installed then); start-up failures appear on the console only.
- The proxy adds one reflective call per use case: negligible next to a SQL round trip.

## Interview one-liner
"Spring Boot 4.1 has gRPC built in, so the server is a bean, not plumbing. Telemetry is Micrometer Observation exported by the OpenTelemetry SDK, with no agent: HTTP, gRPC, JDBC and every use case show up as one trace, with logs and RED metrics on the same trace id, for about 280 MB of RSS."
