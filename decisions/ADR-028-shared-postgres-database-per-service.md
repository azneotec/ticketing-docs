# ADR-028: One shared Postgres container, one database per service

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning round 8; built in session 005)

## Context
The architecture is *database per service*: each service owns its data, and no service reads another's tables; they talk through APIs and events. The usual way to run that is one database server per service. On a 16 GB laptop (ADR-027), four or more Postgres containers cost about 1 GB more than one, and each sits mostly idle. We need the ownership boundary without paying for a server per service.

## Decision
- Locally, run **one Postgres container** (`postgres:18.6`, 512 MB limit, small `shared_buffers`/`work_mem`) in the `ticketing-platform` `core` profile.
- Inside it, **one database and one login per service** (`catalogue`, `inventory`, `booking`, `payment`), created by an init script. Each login **owns** its database (so it manages its own schema with Flyway), and **`CONNECT` is revoked from `PUBLIC`** on every database, including the admin `postgres` database. A login can only connect to its own database; the smoke test proves `catalogue` is refused on `inventory` (`permission denied for database`).
- Services use their own login and never the superuser. Each service still gets its own connection string, so moving a database to its own server later changes configuration, not code.
- `wal_level=logical` is set now, for the Debezium CDC work in Phase 3.

## Alternatives considered
- **One Postgres container per service** — true isolation of failures and resources, the production shape, but ~1 GB more memory locally for no learning gain now.
- **One database, a schema per service** — cheaper still, but schemas in one database are easy to join across by accident, and `CONNECT`-level isolation is lost; the boundary would depend on discipline.
- **One database and one login for everyone** — no isolation at all; would hide accidental coupling until it is too late.

## Consequences
**Good**
- About 1 GB saved locally; Postgres measured at ~20 MB idle.
- Ownership is enforced by Postgres privileges, not by convention: a service cannot even open a connection to another service's database.
- Same code and per-service connection strings as with separate servers, so Kubernetes can split them later.

**Bad (costs we accept)**
- **Shared failure domain:** if the container dies, every service loses its database at once; a noisy service can slow the others (shared CPU, memory, I/O, `max_connections`).
- One Postgres version and one configuration for all services.
- Cross-database privilege isolation is weaker than separate servers: a superuser or a misconfigured grant breaks it.
- Local only: production would use a server (or managed instance) per service; backups are out of scope locally (ADR-026).

## Interview one-liner
"Database per service is about ownership, not about containers: locally one Postgres holds a database and login per service, with `CONNECT` revoked from everyone else so a service physically can't reach another's data — saving a gigabyte of RAM at the cost of a shared failure domain that I'd split out in production."
