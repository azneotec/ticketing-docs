# ADR-010: REST at the edge, gRPC inside

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning rounds 1 and 4; first contract built in session 004)

## Context
The platform has two kinds of callers. The browser (React microfrontends) calls through the Gateway/BFF. Services call each other synchronously, e.g. Booking asks Catalogue or Inventory for data. Both need a contract, but they have different needs: browsers want plain HTTP + JSON that dev tools, caches and the BFF's cookie handling understand. Service-to-service calls want strict typed contracts, low overhead and generated clients, because the same team owns both sides and changes them often.

## Decision
- **Clients → Gateway/BFF: REST over HTTP + JSON.** gRPC is never exposed to the browser.
- **Service → service (synchronous): gRPC** with Protobuf contracts from `ticketing-contracts` (`ticketing-contracts-grpc`). Packages are versioned (`ticketing.catalogue.v1`); `buf lint` and `buf breaking` run in the build.
- Asynchronous communication uses Kafka events, not gRPC (ADR-030, ADR-016).
- Phase 1 shortcut: the web shell calls Catalogue REST directly until the Gateway/BFF exists (Phase 2).

## Alternatives considered
- **REST everywhere** — one style to learn, but internal contracts would be hand-written OpenAPI or none, with JSON overhead and no generated, typed clients; breaking changes are easier to miss.
- **gRPC everywhere, including the browser (gRPC-Web)** — needs a proxy, does not fit the BFF token-handler pattern (httpOnly cookies, ADR-011), and is harder to debug in browser tools.
- **GraphQL at the edge** — flexible for many client shapes, but we have one web client and a BFF that already tailors responses; it adds a third API style.

## Consequences
**Good**
- Each side uses the tool it is good at: browsers get debuggable JSON, services get typed, generated stubs and HTTP/2.
- Contracts are code in one versioned repo; incompatible changes fail the build (`buf breaking`).
- Learning value: both REST design and gRPC (deadlines, status codes, streaming later).

**Bad (costs we accept)**
- Two API styles to design, document, secure and observe.
- The BFF translates between REST and gRPC.
- gRPC needs tooling to inspect by hand (`grpcurl`, installed in session 001) and server reflection or the `.proto` files.

## Interview one-liner
"REST at the edge because browsers and the BFF speak HTTP + JSON; gRPC between services because we own both ends and want typed generated clients and breaking-change checks — at the price of running two API styles and a translation layer in the BFF."
