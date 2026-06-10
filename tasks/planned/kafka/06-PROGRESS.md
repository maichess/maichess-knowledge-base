# Kafka 06 — Progress log

Working log for `06-match-command-side-and-202.md`. Records design decisions, what is
done, and the exact remaining steps so another session can continue without re-deriving.

## The big picture / key realization

Task 06 switches the match **write entrypoint** from synchronous gRPC to Kafka
commands and returns **202**. The authoritative result arrives over the socket.

**Critical prerequisite found during exploration:** *nothing produces `MatchCreated`.*
Grep across all `.cs/.scala/.ts` shows the only writer of `match.events.v1` is the
task-05 projector itself. Match creation (`MatchService.CreateMatchAsync`, driven by
`MatchCommandConsumer` for the normal path and gRPC `CreateMatch` for bot-vs-bot)
writes the durable doc **directly** via `repository.InsertAsync` and never emits
`MatchCreated`. Therefore the live read model (`match:live:{id}`) is **never seeded**,
and the projector's `MatchProjection.Apply`/`MatchHistoryProjection.Apply` both ignore
`MoveSubmitted`/`MoveValidated` when there is no prior state (`state is null ? null`).

So "close the loop" genuinely requires creation to emit `MatchCreated`. This is in
scope here even though the task body frames the entrypoint as `/moves` + `/resign`.

## Design decisions (committed)

1. **Command side reads the live read model** (`ILiveMatchState`). A cold/missing
   model for a match id means the match is unknown to the write side → `MatchNotFoundException`
   (REST 404). Tests inject state via `MatchServiceContext.SetupLiveState`.
2. **Pure command builder** `Kafka/MatchCommands.cs` turns `(LiveMatchState, userId,
   move?)` into the `MatchEvent` to produce, throwing the existing exceptions
   (`NotParticipantException`, `NotYourTurnException`, `MatchAlreadyEndedException`,
   the draw exceptions) on invalid intent. Fully unit-tested. Mirrors `MatchProjector`'s
   envelope/seq conventions: `sequence = state.Sequence + 1`, `correlation_id = newId()`,
   `causation_id = ""`, `producer = "match-manager-service"`.
   - `SubmitMove`  → `MoveSubmitted{move_uci, by, fen, position_history}`
   - `Resign`      → `MatchEnded{status=loser, RESIGNATION}`
   - `OfferDraw`   → `DrawOffered{by}` (opponent-bot ⇒ NotParticipant; existing pending ⇒ DrawOfferAlreadyPending)
   - `AcceptDraw`  → `MatchEnded{DRAW, DRAW_AGREEMENT}` (no pending ⇒ NoDrawOfferPending; self-accept ⇒ NotDrawRecipient)
   - `DeclineDraw` → `DrawDeclined{by}` (no pending ⇒ NoDrawOfferPending)
3. **Producer seam** `Events/IMatchEventProducer.cs` + `Events/KafkaMatchEventProducer.cs`
   ([ExcludeFromCodeCoverage] glue): single, non-transactional, idempotent produce to
   `match.events.v1` keyed by `aggregate_id`, Protobuf serde via `ProtobufEventSerdes`.
4. **Resign/accept-draw emit `MatchEnded` directly** (deterministic — no validator
   round-trip needed). Draw offer/decline emit `DrawOffered`/`DrawDeclined`.
5. **Projector expansion** (`MatchProjector.Decide`): a *consumed* `MatchEnded`,
   `DrawOffered`, `DrawDeclined` (i.e. command-originated, not self-emitted — self-emitted
   ones are deduped by `consumed.Sequence <= state.Sequence`) produces the matching
   socket push (`match_ended`/`draw_offered`/`draw_declined`) and the match-end side
   effects already in `WriteThrough`. `LiveMatchState` gains `PendingDrawOffererUserId`;
   `MatchProjection` folds `DrawOffered`/`DrawDeclined` into it.
6. **Timeout** (`Services/TimeoutWatchdog` + `MatchService.EnforceTimeoutsAsync`):
   replace the Mongo-poll + direct broadcast with a scan of ongoing matches in the live
   read model; for any past `LastMoveAtMs + remaining`, emit exactly one
   `MatchEnded{TIMEOUT}` (status = the side that flagged loses) via the producer. No
   direct socket broadcast — the projector handles the `MatchEnded` push (decision 5).
7. **Creation emits `MatchCreated`** (`CreateMatchAsync`): stop the direct
   `repository.InsertAsync` + `TriggerBotMoveIfNeeded`; emit `MatchCreated` so the
   projector seeds the read model, inserts the durable doc (`WriteThrough`), and kicks
   the first bot move (`OnCreated`). gRPC `CreateMatch` returns an **in-memory** doc
   built from the request inputs (it has everything it minted) — no DB read.
   - External matches (`source=external`) still go through `SyncExternalMatch` and keep
     their current synchronous path (not part of the event loop).
8. **RecordMatchResult gap:** the event-loop match-end path does **not** record player
   ratings/stats — that is task 08 ("rating events; retire RecordMatchResult"). The
   synchronous `RecordMatchResultsAsync` is removed from the move/resign/draw/timeout
   paths here, leaving an interim gap closed by 08. Noted in CONTRACT_NOTES.

## Status — COMPLETE

Task 06 is fully implemented. Build + **283 tests** green; all new/changed included code is
100% line+branch (only the four pre-existing baseline files from task 05 remain partial).
Command side + creation seeding + projector expansion + timeout + REST 202 + gRPC removal +
DI + contract/ADR + client all done. See the service `CONTRACT_NOTES.md` "Match command side
+ 202 (Kafka task 06) — DONE" for the full landed list. The checklist below is the original
plan, kept for history.

**Originally landed first (additive foundation) — build + all 350 tests green (324 baseline + 26 new):**

- [x] Exploration + design (this doc).
- [x] `LiveMatchState.PendingDrawOffererUserId` (additive optional param).
- [x] `Kafka/MatchCommands.cs` + `MatchCommandsTests.cs` (26 tests, 100% covered).
- [x] `Events/IMatchEventProducer.cs` + `Events/KafkaMatchEventProducer.cs` (excluded glue;
      added to stryker-config + service CLAUDE.md exclusions).

These are **additive and non-wired** — `MatchService` does not consume `MatchCommands`/
the producer yet, so the existing synchronous path is untouched and all prior tests pass.
The remaining items below are the actual cutover and will rewrite the synchronous-path
tests. See CONTRACT_NOTES.md "Match command side + 202 (Kafka task 06)" for the same list.

**Remaining (not started):**

- [ ] Rewrite `MatchService` write methods (MakeMove/Resign/OfferDraw/AcceptDraw/DeclineDraw)
      to load live state + produce; remove sync validate/bot/broadcast/record. Inject
      `IMatchEventProducer` into ctor (touches `MatchServiceContext`).
- [ ] Rewrite `EnforceTimeoutsAsync` + `TimeoutWatchdog`.
- [ ] `CreateMatchAsync` emits `MatchCreated`; gRPC `CreateMatch` builds in-memory response.
- [ ] Projector expansion (MatchEnded/DrawOffered/DrawDeclined pushes + pending-offerer state).
- [ ] REST `/moves` + `/resign` (+ draw endpoints) → 202; `MatchesEndpoints` adjustments.
- [ ] gRPC `MatchesGrpcService` MakeMove/ResignMatch — decide: keep sync (for internal
      callers) or remove (task 09 removes the RPCs). Task says removal is 09; here just
      stop the *synchronous in-process work*. The gRPC MakeMove/Resign currently return
      a Match; if the service no longer mutates synchronously they can't. **Open question
      for continuation** — likely make them produce + return the pre-move doc, or defer.
- [ ] Program.cs DI: register `IMatchEventProducer`.
- [ ] Test suite rewrite (MakeMove/Resign/Draw/Timeout/RecordMatchResultOnEnd features +
      steps + context) to assert produced events instead of mutated docs. 100% coverage.
- [ ] Contract: `rest/match-manager.md` /moves + /resign → 202 + client-contract note.
- [ ] Client: optimistic UI already exists; verify `npm run build`/`lint`.
- [ ] Knowledge: mark 202 contract live in `event-driven-architecture.md`.
- [ ] CONTRACT_NOTES: RecordMatchResult interim gap; any blockers.

## Useful coordinates

- Proto: `maichess-api-contracts/protos/events/v1/match_events.proto` (MatchEvent oneof,
  Player, MatchStatus, EndReason, GameResult). `match_commands.proto` has SubmitMove/
  Resign/OfferDraw/AcceptDraw/DeclineDraw commands but the **event loop rides
  match.events.v1**, so the command side emits *events* (MoveSubmitted/MatchEnded/Draw*),
  not the `match.commands` messages — matches the task's "emit the corresponding event".
- Projector pure logic: `Kafka/MatchProjector.cs`; folds `Kafka/MatchProjection.cs` +
  `Kafka/MatchHistoryProjection.cs`; live shell `Events/MatchEventProjectorConsumer.cs`.
- Existing socket-push builder pattern: `MatchProjector.Push/MoveMade/MatchEndedPush`.
- Coverage exclusions list lives in the service `CLAUDE.md` + `stryker-config.json`.
</content>
</invoke>
