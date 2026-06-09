# Kafka 03 — Move-validator stream processor

> Read the [README](README.md) and the match-flow in
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md).
> **Depends on:** `01` (proto + Scala serde). Independent of `04`/`05` (can run in parallel).

## Goal

Make **move-validator** a stateless stream processor in the move loop: consume `MoveSubmitted` from
`match.events.v1`, run its existing pure ZIO chess logic, and produce `MoveValidated` or
`MoveRejected` back to `match.events.v1`. No Kafka code exists in this service yet (CONTRACT_NOTES
only).

## What exists to reuse
- Pure rules engine: `rules/` (Board, MoveApplicator, LegalityFilter, WinConditionDetector),
  `service/ValidatorServiceLive.scala`. **The validation logic is already pure — wrap it, don't
  rewrite it.**
- gRPC server `grpc/MovesServiceImpl.scala` — **keep** the query RPCs (SAN lookups used by analysis);
  this task only **adds** the stream path.

## Implementation
- Add **zio-kafka** + the ScalaPB Protobuf serde (from `01`) — see `build.sbt`/`project/plugins.sbt`.
- A consumer of `match.events.v1` filtered to `MoveSubmitted`: for each, apply the move to
  `MoveSubmitted.fen` via the existing applicator/legality logic; produce:
  - `MoveValidated{resulting_fen, game_result, position_history}` on success, or
  - `MoveRejected{move_uci, reason}` on an illegal move.
- **Carry the opaque `position_history` blob through unchanged** — the validator owns it (threefold
  repetition); Match Manager never inspects it. Compute the new `game_result` (checkmate/stalemate/
  50-move/threefold/insufficient) using the existing detectors.
- Envelope: copy `aggregate_id` (matchId); set `causation_id` = the `MoveSubmitted.event_id`;
  increment `sequence` per the program's ordering rule. Pure + idempotent → safe to reprocess.
- Wrap the consume→produce in a Kafka transaction for effectively-once (see README).
- Keep the I/O (consumer/producer wiring) in a thin layer; the decision logic stays pure and tested.

## Contract changes
None beyond `01` (the `match.events` proto already defines these payloads).

## Tests (Scala, ZIO-test; mirror `stryker4s.conf` exclusions)
- Pure: legal move → `MoveValidated` with correct `resulting_fen`/`game_result`; illegal →
  `MoveRejected` with reason; each terminal result type detected; `position_history` passed through.
- Serde round-trip for `MoveSubmitted`/`MoveValidated`/`MoveRejected`.
- The stream wiring (I/O) excluded from scoverage like other live-dependency code; the transform is covered.
- `sbt test`; `sbt stryker` to audit the transform mutants.

## Verify
- Locally drive a `MoveSubmitted` onto the topic → observe the matching `MoveValidated`/`MoveRejected`.
- Query RPCs still serve analysis unchanged.

## Docs to update
- `move-validator-position-history` (knowledge/services) — note the validator now also runs as a
  stream processor and where the position_history blob rides.
- This service's `CONTRACT_NOTES.md` — remove the "no Kafka code" note.
