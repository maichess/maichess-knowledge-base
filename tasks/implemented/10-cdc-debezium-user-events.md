# 10 — Caching Stage 2: close the user dual-write with Debezium CDC

> Read `conventions.md`,
> [change-data-capture.md](../../knowledge/architecture/change-data-capture.md), and
> [event-driven-architecture.md](../../knowledge/architecture/event-driven-architecture.md) first.
> **New infrastructure:** Kafka Connect + Debezium (deployment is part of this prompt).
> This must land **before** Stage 3 (`11`), which trusts `user.events`.

## Goal

Eliminate the **dual write** in the User service (write `user-db` *and* emit `user.events` —
non-atomic) by deriving change events from the Postgres WAL with **Debezium CDC**, so the event
stream and the database can no longer diverge.

## The rule (do not violate)

Per [change-data-capture.md](../../knowledge/architecture/change-data-capture.md): **CDC/outbox for
CRUD master-data stores; application-level event sourcing for the match aggregate. Never both on
one store.** Do **not** put Debezium on `match.*`.

## Implementation

1. **Postgres for logical replication.** Configure `user-db` Postgres with `wal_level=logical`
   and a publication/replication slot for the `users` table (and any other user-db tables that
   feed events).
2. **Debezium Postgres connector** → emits raw change events to **`user.cdc.v1`**
   (keyed by userId). Connector config lives as a deployable resource (see Deployment).
3. **Curate `user.events.v1` from CDC.** `user.events.v1` stays the public, compacted,
   envelope-wrapped contract. Feed it from `user.cdc.v1` via a transform (Kafka Connect SMT/
   single-message transform, or a small stream processor) that maps CDC rows → the existing
   `user.events` envelope + payload. Run this **side by side** with the current User-service
   emitter and reconcile (compare outputs) before cutover.
4. **Remove the dual write.** Once `user.events` from CDC matches the legacy emitter, delete the
   User service's event-emit side effect. The User service now writes Postgres **only**; the
   commit (WAL) is the single source of the change event.

> Keep the `user.events` **schema/envelope unchanged** so Stage 3 and existing consumers are
> unaffected — this stage changes *how* the topic is produced, not its contract.

## Deployment (already scaffolded — verify, don't rebuild)

The Helm side is in place, gated behind `kafkaConnect.enabled`:

- **Kafka Connect + Debezium** (`templates/kafka-connect.yaml`) with the **user-cdc** connector
  (Postgres WAL → `user.cdc.v1`) registered by a post-install Job; Connect's internal
  `config`/`offset`/`status` topics auto-create with the right replication factor.
- **Postgres `wal_level=logical`** + pinned `max_wal_senders`/`max_replication_slots`
  (`templates/postgres.yaml`, gated). The `maichess` superuser has REPLICATION; Debezium
  auto-creates the slot and a `filtered` publication for `public.users`.
- `user.cdc.v1` topic (`values.yaml` `kafka.topics`): 7d retention, `delete` cleanup — raw
  stream, **not** a public contract.

Your job here: enable it (`kafkaConnect.enabled=true`), `helm lint` + `helm template` to confirm,
deploy, and verify the connector reaches `RUNNING` (`GET kafka-connect:8083/connectors/user-cdc/status`).
Adjust `kafkaConnect.postgres.*` in `values.yaml` if your DB name/user differ from `maichess`.

## Contracts

- `user.events.v1` schema is unchanged — no `maichess-api-contracts` bump needed unless the
  envelope mapping reveals a gap (if so, follow the standard publish/bump handoff).
- Record `user.cdc.v1` as an **internal** stream in the events docs, explicitly *not* a consumer-
  facing contract.

## Tests (mandatory)

- Test the CDC→`user.events` transform: a representative set of Postgres change rows
  (insert/update of rating, stats, username, dev_mode) maps to the correct `user.events`
  envelopes + payloads. 100% on non-excluded transform code.
- Reconciliation check: legacy-emitter output vs CDC-derived output agree for the same operations
  (can be a test harness / one-shot job, documented).

## Verify

1. Kill the User service mid-operation in a test env; confirm `user.events` still reflects the
   committed Postgres state (no lost/duplicated event) — the dual-write failure mode is gone.
2. A rating/stat/username/dev_mode change flows Postgres → `user.cdc.v1` → `user.events.v1` with
   the unchanged envelope.
3. Existing `user.events` consumers are unaffected.

## Checklist

- [ ] Read the CDC + event-driven ADRs.
- [ ] Postgres logical replication enabled; slot/publication provisioned.
- [ ] Debezium connector → `user.cdc.v1`; `user.events` curated from it; reconciled side-by-side.
- [ ] User-service event-emit dual write removed (Postgres-only write).
- [ ] Kafka Connect + connector + internal topics deployed in `maichess-deploy`.
- [ ] Transform tests + reconciliation; coverage verified.
- [ ] `user.cdc.v1` documented as internal in the events docs.
