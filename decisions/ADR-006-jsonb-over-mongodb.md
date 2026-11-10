# ADR-006: Postgres JSONB instead of MongoDB for seat maps

- **Status:** Accepted
- **Date:** 2026-11-10 (decided in planning, "Stores" row of the plan; built in session 006)

## Context
A seat map is a tree: sections → rows → seats. It is read and written as a whole together with its event, it varies a lot between venues (a stadium vs a theatre), and it is rarely queried by its inside. That shape suggests a document store such as MongoDB. On the other hand, catalogue data is also relational (events belong to venues, lists are filtered by status and date), we already run one shared Postgres for every service (ADR-028), and the laptop has 16 GB in total (ADR-027).

## Decision
We store each event's seat map as one **`jsonb` column** (`event.seat_map`) in Postgres, next to the relational columns:
- JSON shape is owned by the persistence adapter (`SeatMapDocument` records), not by the domain records, so a domain rename can't silently change stored JSON (ADR-031).
- A **GIN index with `jsonb_path_ops`** on `seat_map` supports containment queries (`seat_map @> '{"sections":[{"name":"Balcony"}]}'`).
- Relational fields (status, start time, venue) stay real columns with a B-tree index on `(status, starts_at)` and a foreign key to `venue`.
- Data read back from JSONB goes through the domain constructors again, so a corrupt document fails loudly.

## Alternatives considered
- **MongoDB** — a natural fit for the tree. Rejected: one more database to run, secure and back up on a 16 GB laptop; it would lose the foreign key to venues and the single transaction with the event row; and Postgres JSONB covers what we need. (Interview angle: "pick the boring store until it hurts".)
- **Fully normalised tables (`section`, `row`, `seat`)** — gives row-level constraints, but a 50,000-seat stadium becomes 50,000 rows to insert and join per event, for data that's always read whole. The authoritative per-seat *state* (holds, sold) is Inventory's job in Phase 2, not Catalogue's.
- **`json` (text) instead of `jsonb`** — keeps formatting and key order but can't be indexed with GIN and is re-parsed on every query.

## Consequences
**Good**
- One database, one transaction for event + seat map, and the foreign key to the venue is kept.
- Flexible layouts without schema migrations for each venue shape.
- Containment queries are indexable. The test `jsonbContainmentFindsEventsBySectionName` shows one.

**Bad (costs we accept)**
- The database doesn't enforce the shape inside the document (no "seat numbers unique per row" constraint). The domain enforces it on write and again on read.
- Changing the JSON shape later needs a data migration (`jsonb_set` / an update script) as well as code.
- JPQL can't query inside JSONB; those queries are native SQL tied to Postgres.
- Every update rewrites the whole document (Postgres rows are immutable versions), which is fine for small, rarely changed seat maps.

## Interview one-liner
"Seat maps are a document, so I stored them as JSONB in the Postgres we already run instead of adding MongoDB. We keep transactions and the venue foreign key, a GIN index makes containment queries fast, and the domain validates the shape because the database can't."
