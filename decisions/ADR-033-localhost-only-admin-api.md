# ADR-033: Localhost-only admin API until Phase 2

- **Status:** Accepted (temporary; to be superseded when Keycloak arrives, ADR-011)
- **Date:** 2026-11-11 (decided in planning rounds 13 and 14; built in session 007)

## Context
Organisers create venues and events and publish them through `/admin/**` in `catalogue-service`. Real authentication (Keycloak, the BFF token handler, JWT validation in services, ADR-011) is Phase 2 work. Phase 1 still needs the admin API to load data and to prove the gate. An unauthenticated write API is a risk the moment the service is reachable from anywhere but the developer's own machine.

## Decision
- `/admin/**` has **no authentication until Phase 2**, but it accepts **only loopback callers** (`127.0.0.0/8`, `::1`). Everyone else gets **403** with a Problem Details body (`urn:ticketing:problem:admin-localhost-only`).
- Enforcement: `LocalhostOnlyAdminInterceptor`, a Spring MVC interceptor registered for `/admin` and `/admin/**`.
  - It checks the **TCP peer address** (`request.getRemoteAddr()`), parsed with `InetAddress.ofLiteral` (no DNS).
  - `X-Forwarded-For` is ignored: Spring Boot applies forwarded headers only when configured, and we don't configure it, so a caller can't claim to be localhost. A test proves it.
  - An interceptor rather than a servlet filter: it is matched by the same path patterns as the controllers, so no path spelling (`//admin`, `/admin;x`) can reach an admin controller without passing it. The intent ("loopback only, 403 otherwise") is the one decided in round 14.
- **No CORS for `/admin`**, so a web page in the browser can't call it.
- Locally every platform port binds to `127.0.0.1` (ticketing-platform rules); the service itself listens on all interfaces, so the guard, not the bind address, is the protection. That keeps it effective inside a container too, where calls from other containers are not loopback.
- **Replacement (Phase 2):** Keycloak roles (`organiser`) checked by the gateway (coarse) and by the service (JWT validation). The interceptor is then deleted and this ADR superseded.

## Alternatives considered
- **Authenticate now (Keycloak or basic auth)** — Keycloak is planned for Phase 2 with the BFF; adding it now drags the auth design forward. Basic auth would be throw-away work with its own secret handling.
- **Bind the whole service to `127.0.0.1`** — protects admin, but also hides the public API and gRPC from other containers and from the browser via the gateway later.
- **A separate admin port** (management-style) — cleaner separation, but more config and a second server for one Phase 1 shortcut.
- **No admin API** (load data with SQL scripts) — skips the REST design, the `If-Match` flow and the gate's admin flow.

## Consequences
**Good**
- The admin flow works now with `curl`, for the Phase 1 gate and for loading data into the web shell.
- An accidental exposure (a port opened on the LAN, a container on a shared network) gives 403s, not an open write API.
- The boundary is one class with tests, easy to delete when Keycloak arrives.

**Bad (costs we accept)**
- Anything running on the same machine is trusted, including any local process or a compromised browser extension making requests from localhost.
- Behind a reverse proxy every request would look like the proxy's address; if the proxy is local, admin would be open to the world. Phase 2's gateway must come with real auth, not this guard.
- Swagger UI and the OpenAPI document are public for now (read-only, but they describe the admin API).

## Interview one-liner
"Until Keycloak arrives, the admin API is guarded by network position: an interceptor that trusts only the TCP peer address and accepts loopback, so `X-Forwarded-For` can't fake it. It's a deliberate, documented stopgap with a named replacement, not security we'd ship."
