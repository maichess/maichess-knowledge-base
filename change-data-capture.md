# Change Data Capture & the Outbox Boundary

**Status:** Accepted — implementation in `feature-prompts/10`
**Relates to:** [event-driven-architecture.md](event-driven-architecture.md),
[caching-and-read-models.md](caching-and-read-models.md)

## Context

`user.events.v1` is produced by the User service *and* the same operations write `user-db`
through DatabaseService. That is a **dual write**: two non-atomic side effects. If the process
dies between them, the event log and the database diverge silently — and every read model,
cache, and search index downstream of `user.events` inherits the divergence. Stage 3 of the
caching work (user read replicas) *trusts* that topic, so the dual write must be closed first.

CDC and event sourcing solve the same problem — "get a reliable stream of what changed" — by
opposite means. The match aggregate already solves it at the application layer (it is genuinely
event-sourced). The CRUD-mastered stores (user-db, and the Mongo collections written as plain
CRUD) do not.

## Decision

**Use Debezium (Kafka Connect) CDC to derive change events from the CRUD-mastered stores.
Keep application-level event sourcing for the match aggregate. Never apply both to the same
store.**

- **user-db (Postgres):** a Debezium Postgres connector tails the WAL and emits change events to
  `user.cdc.v1`. The User service stops dual-writing; it writes Postgres only, and the canonical
  `user.events.v1` is produced by transforming the CDC stream (or, equivalently, `user.events` is
  fed from `user.cdc.v1`). The write becomes atomic by construction — the WAL *is* the commit.
- **match-db (Mongo):** a Debezium Mongo connector tails the oplog for the collections that need
  to feed derived read models (search indexes, see
  [search-elasticsearch.md](search-elasticsearch.md)) without their owning service dual-writing.
- **match aggregate:** unchanged. It stays event-sourced at the app layer. Putting Debezium on
  `match.*` would double-source events we already produce — explicitly forbidden.

CDC taps the databases **underneath** the DatabaseService gRPC abstraction (Postgres WAL, Mongo
oplog). That is intended: CDC is infrastructure plumbing, not an application concern, and it does
not violate the "only adapters touch the DB" rule because it is not application code.

### The rule, stated once

> Application-level events for the genuinely event-sourced aggregate (match).
> Debezium CDC (or a transactional outbox) for master-data CRUD stores (user, and CRUD Mongo
> collections). Exactly one change-event source per store — never both.

If a future service is a new system of record, it gets a CDC connector or an outbox, not a
hand-rolled dual write.

## Topics

| Topic | Source | Key | Cleanup |
|---|---|---|---|
| `user.cdc.v1` | Debezium / Postgres `users` | userId | delete (bounded retention) |
| `match.cdc.v1` | Debezium / Mongo `matches`, `analysis_games` | document id | delete |

`user.events.v1` remains the **public, curated** contract (compacted, envelope-wrapped);
`*.cdc.v1` are **raw** change streams consumed by a transform/relay, not by feature services
directly. This keeps the published event contract stable while CDC handles delivery.

## Deployment

- New Helm components: a **Kafka Connect** cluster and the **Debezium** Postgres + Mongo
  connectors (connector config as Kubernetes resources / Connect REST calls in an init job).
- Postgres must run with `wal_level=logical` and a replication slot/publication for the
  connector; Mongo must expose an oplog (replica set). Document both in `maichess-deploy`.
- Connector offsets/status topics are internal Connect topics — provision them in the topic-init
  job alongside the existing event topics.

## Migration order

1. Stand up Kafka Connect + the Postgres connector; emit `user.cdc.v1` (no consumer yet).
2. Feed `user.events.v1` from `user.cdc.v1`; run side by side with the existing producer and
   reconcile.
3. Remove the User service's event-emit dual write; Postgres is now the only synchronous write.
4. Add the Mongo connector when the search indexer (`feature-prompts/13`) needs it.
