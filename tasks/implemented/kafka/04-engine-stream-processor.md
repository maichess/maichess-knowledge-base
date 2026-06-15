# Kafka 04 — Engine stream processor

> Read the [README](README.md) and the match-flow in
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md).
> **Depends on:** `01` (proto + Scala serde). Independent of `03`/`05`.

## Goal

Make **engine** a stream processor for bot moves: consume `BotMoveRequested` from `match.events.v1`,
compute the best move with its existing search, and produce `BotMoveCalculated`. No Kafka code exists
in this service yet (CONTRACT_NOTES only).

## What exists to reuse
- The engine/search stack (`service/EngineServiceLive.scala`, `chess/` search variants, bot registry).
  **Reuse `GetBestMove`'s logic** — this task adds a streaming entrypoint, it does not change search.
- gRPC server `grpc/BotsServiceImpl.scala` — **keep** `ListBots` (a query RPC). `GetBestMove` gRPC is
  removed later in `09` once nothing calls it.

## Implementation
- Add **zio-kafka** + the ScalaPB Protobuf serde (from `01`).
- Consume `match.events.v1` filtered to `BotMoveRequested{fen, bot_id, time_limit_ms, request_id}`;
  compute the move; produce `BotMoveCalculated{move_uci, evaluation_cp, request_id}`.
- **Dedupe on `request_id`** — bot-move calculation is nondeterministic, so it is not safe to
  reprocess. Keep a bounded seen-set / use the request_id as the idempotency key so a redelivery does
  not double-submit. (The validator/projector use Kafka transactions; the engine's guard is
  `request_id`.)
- Envelope: copy `aggregate_id` (matchId); `causation_id` = `BotMoveRequested.event_id`.
- Keep the search logic pure/separate from the consumer/producer I/O.

## Contract changes
None beyond `01`.

## Tests (Scala, ZIO-test; mirror `stryker4s.conf`)
- A `BotMoveRequested` yields a `BotMoveCalculated` with a legal move + the same `request_id`.
- Dedupe: a redelivered `request_id` does not produce a second `BotMoveCalculated`.
- Serde round-trip; stream wiring excluded from scoverage, the decision/dedupe logic covered.
- `sbt test`; `sbt stryker` on the dedupe + move-selection logic.

## Verify
- Drive a `BotMoveRequested` → observe one `BotMoveCalculated`; a duplicate request_id → still one.
- `ListBots` gRPC unchanged.

## Docs to update
- This service's `CONTRACT_NOTES.md` — remove the "no Kafka code" note.
