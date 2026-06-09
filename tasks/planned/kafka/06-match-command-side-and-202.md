# Kafka 06 — Match command side + 202 + close the loop

> Read the [README](README.md) and the match-flow + client-contract sections of
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md).
> **Depends on:** `05` (read model + projector); `03`/`04` (validator + engine) for the loop to close.

## Goal

Switch the match write entrypoint from synchronous gRPC to commands: `POST /moves` and `/resign`
validate against the Redis read model, emit `MoveSubmitted`/intents, and return **202 Accepted** — the
authoritative result arrives over the socket. This closes the move loop (command → validator →
projector → engine → projector) and retires the synchronous in-process move path.

## What exists to reuse
- `MatchService.MakeMoveAsync`/`ProcessBotMoveAsync` (the current synchronous path) — replace with
  command emission; keep participant/turn-derivation logic, now checked against the Redis read model
  from `05`.
- `Services/TimeoutWatchdog.cs` (currently polls Mongo + broadcasts directly) — replace.
- The client already does optimistic UI + socket confirmation, so the 202 move is mostly server-side.

## Implementation
- **`POST /matches/{id}/moves`:** check participant + turn against the Redis read model; produce
  `MoveSubmitted{move_uci, by, fen, position_history}` to `match.events.v1`; return **202 Accepted**
  (no body / minimal ack). The projector + validator from `03`/`05` carry it to `move_made`.
- **`POST /resign`** (and draw offer/accept/decline): emit the corresponding command/event
  (`ResignCommand` / `OfferDraw` / `AcceptDraw` / `DeclineDraw`); projector ends or updates the match.
  These also return **202**.
- **Timeouts:** replace the polling watchdog with a per-match scheduled check keyed on
  `last_move_at + remaining`; on expiry emit `MatchEnded{TIMEOUT}` (a new timer component in
  match-manager, per the ADR). No direct broadcast — the projector handles `MatchEnded`.
- Remove the now-dead synchronous validate/bot-move/broadcast code in `MatchService` (the actual gRPC
  client *removals* — internal `MakeMove`, `Engine.GetBestMove`, validator `ValidateMove` — happen in
  `09`; here, stop calling them).

## Contract changes
- `rest/match-manager.md`: `POST /moves` and `/resign` change from **200 + full match** to **202
  Accepted**. Record the client-contract change (optimistic UI + socket `move_made`/`match_ended`).
- `matches.proto`: if the internal `MakeMove` RPC is referenced, leave it for `09` to remove; no new
  proto needed (commands ride `match.events`/`match.commands` from `01`).
- Follow the contract-publish handoff if anything under `maichess-api-contracts/` changes.

## Tests (Reqnroll + xUnit)
- `POST /moves` by the right player on their turn → `MoveSubmitted` produced + **202**; wrong
  player/turn → 4xx without producing; `/resign` and draw intents emit the right command + 202.
- Timeout: a match past `last_move_at + remaining` emits exactly one `MatchEnded{TIMEOUT}`.
- End-to-end (with `03`/`04`/`05` wired): a submitted human move comes back as `move_made`; a bot
  reply flows `BotMoveRequested → BotMoveCalculated → MoveSubmitted → move_made`.
- 100% non-excluded; endpoint adapters excluded as usual.
- Client: `npm run build`/`lint` + manual click-through (move shows optimistically, confirmed by socket).

## Verify
- Staging: play a full human-vs-human and a human-vs-bot game over the event loop; moves and end
  states arrive via socket; reads come from the Redis read model; no synchronous validate/bot gRPC on
  the move path; timeout ends a stalled game.

## Docs to update
- `event-driven-architecture` — mark the 202 client contract live.
- `match-history-and-stats` if the write path notes change.
