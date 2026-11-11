# ADR-032: REST errors as Problem Details; optimistic locking via ETag / If-Match

- **Status:** Accepted
- **Date:** 2026-11-11 (decided in planning round 14; built in session 007)

## Context
`catalogue-service` is the first REST API. Its errors will be read by the web shell now and by the BFF in Phase 2, so they need one predictable shape, not ad-hoc JSON per endpoint. The domain already reports rule violations as a sealed `DomainException`, and the application reports `NotFoundException` and `ConcurrentUpdateException`. Events are edited by organisers. Two organisers, or one with two browser tabs, must not silently overwrite each other's changes. The database already has a version column (ADR-031), but over HTTP the read and the write are separate requests, so the version the client saw has to travel with the request.

## Decision
- **Every error is a Problem Details body** (RFC 9457, `application/problem+json`) with a stable `type` URN (`urn:ticketing:problem:<slug>`), `title`, `status`, `detail` and `instance`. Validation errors add an `errors` list with every failing field. Spring MVC's own errors (unreadable JSON, wrong parameter type) use the same shape through `ResponseEntityExceptionHandler`.
- **Exceptions are mapped in one place** (`ProblemDetailsAdvice`). An exhaustive switch over the sealed `DomainException` (no `default`) means a new domain exception does not compile until it has a status. Statuses: invalid value 400, not found 404, illegal transition 409, stale version **412**, missing `If-Match` **428**.
- **The event's version is its `ETag`** (`"3"`). Reads return it; `If-None-Match` gives 304.
- **State-changing admin calls require `If-Match`.** `PublishEvent` takes the expected version, so the service rejects a stale one before changing anything. The repository's version check still catches a race between that check and the save.
  - Missing `If-Match` → 428 Precondition Required.
  - `*` → 428 too: it matches any version, which would switch the protection off.
  - Malformed (not one strong ETag) → 400.
  - Stale → 412 Precondition Failed.
- **412 for a stale version, not 409.** RFC 9110 defines 412 as "a precondition you sent is false". Because every write carries `If-Match`, a version conflict is always exactly that. 409 stays for business-rule conflicts. A client can then tell "reload and retry" (412) from "this will never work" (409).

## Alternatives considered
- **Custom error JSON** (`{"error": "...", "code": 123}`) — every team invents one, clients must learn each. Problem Details is a standard Spring supports natively.
- **409 for every conflict** — common and simpler, but it mixes "your copy is stale, retry" with "the state forbids this", which need different client behaviour.
- **Optional `If-Match`** (check only if present) — friendlier for quick `curl` calls, but a client that forgets the header silently loses lost-update protection. 428 makes forgetting loud.
- **Version in the request body or URL** — works, but re-invents what HTTP's conditional requests already define, and caches and tools understand `ETag`.
- **Pessimistic locking** (lock while editing) — needs lock expiry and holds rows; edits are rare and short, so optimistic is the better fit (ADR-031).

## Consequences
**Good**
- One error shape for all clients. The web shell and BFF can switch on `type`, and the OpenAPI spec documents it.
- Lost updates are impossible through the API, and the rule is visible in the HTTP contract, not hidden in the server.
- The compiler forces a decision whenever a new domain exception appears.
- `ETag` + `If-None-Match` gives cheap revalidation for reads.

**Bad (costs we accept)**
- Clients must keep the `ETag` and send it back. Plain `curl` calls need the header.
- Each use case that changes state needs an expected-version parameter (a small port change, made for `PublishEvent`).
- A 500 still uses Spring Boot's default error JSON: catching everything risks leaking internals. To revisit when the BFF exists.

## Interview one-liner
"Errors are RFC 9457 Problem Details, mapped from a sealed exception hierarchy by an exhaustive switch, so a new failure can't ship without a status code. Concurrency control is HTTP-native: the version is the ETag, writes require If-Match (428 without, 412 if stale), and 409 is reserved for genuine business-rule conflicts."
