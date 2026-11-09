# ADR-002: Event ticketing as the domain

- **Status:** Accepted
- **Date:** 2026-11-09 (decided in planning round 2)

## Context
The goal of the project is to experience distributed transactions, sagas and data consistency across services first-hand, inside services built with strict hexagonal architecture. The domain therefore has to create real distributed-systems problems on its own, not ones forced onto it.

## Decision
We build an **event ticketing** platform (option C). Scope: purchase, organiser event cancellation and refunds. A flash-sale waiting room is a later learning item, out of scope for now.

## Alternatives considered
- **A. Order fulfilment** — not chosen.
- **B. Travel booking** — not chosen.
- **C. Event ticketing** — chosen.
- **D. Wallet** — not chosen.

Ticketing was chosen as a fresh domain whose core problems are naturally distributed: seat holds that expire, contention for the same seat, and idempotency (no double booking or double charge).

## Consequences
**Good**
- Natural sagas: hold seat → pay → confirm booking, with compensation when payment fails or a hold expires.
- A hard, testable invariant — no overselling — that drives the Inventory design and has its own SLO (zero oversells).
- Cancellation and refunds give a second, large-scale workflow (refund work queue).

**Bad (costs we accept)**
- Contention and expiry are hard problems that appear early (seat holds in Phase 2).
- The most realistic stress case, flash sales, is deferred, so early load tests do not cover it.

## Interview one-liner
"I chose event ticketing because seat holds with expiry, contention for the same seat and idempotent payments force real distributed-systems problems — sagas, compensation and a no-overselling invariant — instead of artificial ones."
