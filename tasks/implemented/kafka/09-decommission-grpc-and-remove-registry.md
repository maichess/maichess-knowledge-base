# Kafka 09 — Decommission dead gRPC + remove the Schema Registry

> Read the [README](README.md), the "what stays synchronous" list in
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md), and the
> registry-removal phase in
> [serialization-protobuf-migration](../../../knowledge/architecture/serialization-protobuf-migration.md).
> **Depends on:** `02` (live topics on proto) + `06`, `07`, `08` (every command/fact flow moved off
> gRPC). This is the final task — only run it once nothing uses the paths below.

## Goal

Reach the target end state: remove the gRPC RPCs the migration made dead, and remove the Confluent
Schema Registry in favor of **raw Protobuf bytes** with schemas managed solely in
`maichess-api-contracts`.

## Part A — Decommission dead gRPC
Remove (server + clients + proto RPCs), after grepping the workspace to confirm zero callers:
- `Socket.EmitEvent` / `Socket.BroadcastMatchEvent` (socket-service still runs a gRPC server for these).
- match-manager internal `MakeMove`.
- the `Matches.CreateMatch` **client** wiring that's no longer used (the human paths now publish
  `CreateMatchCommand`; **keep** `CreateMatch` only if bot-vs-bot still needs synchronous start_fen
  validation — confirm against `16`/bot-arena before removing).
- `Engine.GetBestMove` (the engine now streams `BotMoveCalculated`).
- the legacy `Socket*Notifier`/`SocketNotifier` gRPC transport branches and the `Socket:Transport`
  flag (once `kafka` is the only path).

**Keep** (the ADR's surviving synchronous surface): all query RPCs — `GetMatch`, `ListMatches`,
legal-moves, `GetUser`, `ListBots`, the validator SAN lookups; `DatabaseService` CRUD; and
`Auth.ValidateToken`.

## Part B — Remove the Schema Registry
Only after **every** topic is Protobuf-only:
- Switch all producers/consumers from the Confluent registry serde to **raw Protobuf** encode/decode
  (generated types from `Maichess.PlatformProtos`; no registry lookup, no magic byte).
- Remove the `schema-registry` Helm component (`maichess-deploy/helm/.../schema-registry.yaml`) and
  every `SCHEMA_REGISTRY_URL` env/config wiring across services and the chart.
- Delete the now-unused Avro artifacts and any remaining `.avsc` (the `events/v1/*.avsc` set) and the
  registry-specific serde packages from each service.

## Tests
- Build/test every touched service green after RPC removal (no dangling references — grep first, per
  the [conventions](../../conventions.md) "finish the job" rule).
- Raw-Protobuf encode/decode round-trip per topic, **without** a registry.
- Coverage bars hold; Stryker/Stryker4s exclusions updated for removed code.

## Verify
- Full end-to-end on raw Protobuf with the registry gone: play a match (move loop), matchmaking, the
  socket push, analysis, a rating update, and a cheat event — all work; the `schema-registry`
  deployment no longer exists; ordering/idempotency/envelope semantics unchanged.
- Grep the workspace for `EmitEvent|BroadcastMatchEvent|GetBestMove|SCHEMA_REGISTRY_URL|\.avsc` → only
  intentional hits remain.

## Docs to update
- `serialization-protobuf-migration` — mark the registry removed; the platform is single-IDL Protobuf.
- `event-driven-architecture` — update the synchronous-surface list to the final kept RPCs.
- `deployment-and-environments` — drop the schema-registry component.
- Program [README](README.md) — mark the program complete.
