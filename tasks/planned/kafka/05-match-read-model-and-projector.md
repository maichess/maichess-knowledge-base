# Kafka 05 — Match read model + projector

> Read the [README](README.md) and the match-flow + read-model sections of
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md) and
> [caching-and-read-models](../../../knowledge/architecture/caching-and-read-models.md).
> **Depends on:** `01` (proto), `03` (`MoveValidated`), `04` (`BotMoveCalculated`).

## Goal

Build match-manager's **projector** and the **Redis live read model** (the CQRS read side for ongoing
matches). This is the half of the move loop that turns validated facts into applied state, drives bot
moves, and serves live reads — without yet changing the write entrypoint (that's `06`).

## What exists to reuse
- Mongo write path via `DatabaseService` (`Data/MatchRepository`, `Services/MatchService`) — keep as
  the **durable** store; the projector materialises history into it.
- Redis is already a dependency for the finished-match cache (`Data/RedisMatchCache.cs`,
  `IMatchCache`) and the user replica (`Data/RedisUserReplica.cs`) — reuse the connection/seam style.
- Clock rules (elapsed subtract, increment on non-terminal) and FEN-history semantics are in
  `MatchService`/CLAUDE.md — reuse the math; move it to the projector.

## Implementation
- **Redis live read model:** a per-match projection — current FEN, clocks (`white_time_ms`/
  `black_time_ms`), turn, move index, and the opaque `position_history` blob. Add `StackExchange.Redis`
  (the `ConnectionStrings__Redis` env is already wired in the chart but unused). Rebuild the projection
  from `match.events.v1` on startup (replay from the log). Behind an `ILiveMatchState` seam (Redis impl
  `[ExcludeFromCodeCoverage]`, the pure projection logic unit-tested).
- **Projector** consuming `match.events.v1`:
  - `MoveValidated` → update read model + compute clocks → produce `MoveApplied` → produce
    `socket.outbound` `move_made`; if `game_result` terminal → produce `MatchEnded` (+ socket
    `match_ended`); else if the side to move is a bot → produce `BotMoveRequested{request_id}`.
  - `BotMoveCalculated` → produce `MoveSubmitted` (the bot's move) → re-enters the validator loop.
  - `MatchCreated`/`MoveApplied`/`MatchEnded` → also materialise durable history into match-db via
    `DatabaseService` (the write-through), and on `MatchEnded` keep the existing finished-match cache
    refresh + page eviction (`OnMatchEndedAsync`).
- **REST live reads** (`GET /matches/{id}`, positions) read the Redis read model for ongoing matches;
  finished matches keep using the existing cache/match-db path.
- Idempotency: dedupe on `(aggregate_id, sequence)`; wrap the consume→produce (projector) in a Kafka
  transaction for effectively-once (see README).

## Contract changes
None beyond `01` (uses `match.events` payloads). REST shape unchanged here (the 202 change is `06`).

## Tests (Reqnroll + xUnit; mirror coverlet/Stryker exclusions)
- Pure projection: applying each event to read-model state (FEN/clocks/turn/index/history); clock
  math incl. increment-on-non-terminal and no-increment-on-terminal; rebuild-from-log reconstructs
  state; terminal `MoveValidated` → `MatchEnded`; bot-to-move → `BotMoveRequested`;
  `BotMoveCalculated` → `MoveSubmitted`.
- Read-model-backed REST reads return the projected state; durable write-through called.
- Redis impl + consumer/producer glue `[ExcludeFromCodeCoverage]`; everything else 100%.
- `dotnet test -p:CollectCoverage=true`; `dotnet stryker` on the projection/clock logic.

## Verify
- Replay a known `match.events` sequence → Redis read model and match-db history match the expected
  board/clocks; `move_made`/`match_ended` socket pushes emitted; a bot turn emits `BotMoveRequested`
  and a returned `BotMoveCalculated` re-enters as `MoveSubmitted`.

## Docs to update
- `caching-and-read-models` — document the live match read model (the piece previously "not
  implemented"); `event-driven-architecture` read-model section if anything firmed up.
