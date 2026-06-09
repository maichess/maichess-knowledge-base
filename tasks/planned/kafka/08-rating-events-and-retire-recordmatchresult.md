# Kafka 08 — Rating events; retire RecordMatchResult gRPC

> Read the [README](README.md), [rating-glicko2](../../../knowledge/domain/rating-glicko2.md), and
> [match-history-and-stats](../../../knowledge/domain/match-history-and-stats.md).
> **Depends on:** `01` (proto), `06` (`MatchEnded` is emitted by the projector).

## Goal

Emit rating/result updates on match end as events instead of a synchronous gRPC call. match-manager
currently calls user-service `RecordMatchResult` over gRPC (`MatchService.RecordMatchResultsAsync`,
one call per human, bot-vs-bot records nobody). Replace that with a `user.events.v1` flow and retire
the RPC.

## Current state
`user.events.v1` exists (CDC-derived for user CRUD per the caching work) but has **no rating
producer/consumer**. The Glicko-2 update lives in user-service `RecordMatchResultAsync`.

## Implementation
- **Producer:** when the projector emits `MatchEnded` (`05`/`06`), also emit a rating-relevant event
  for each human participant — either a dedicated `match.events` `MatchEnded` consumer in user-service,
  or a `MatchResultRecorded`/rating-update event the rating side consumes. Choose the path that keeps
  `user.events.v1` as the user fact stream (align with the CDC rule in
  [change-data-capture](../../../knowledge/architecture/change-data-capture.md): event-sourcing for
  match, CDC/outbox for the user CRUD store — don't double-write).
  - Recommended: user-service (or the rating owner) **consumes `MatchEnded`** from `match.events.v1`,
    runs the existing Glicko-2 update, and the resulting rating change flows out on `user.events.v1`
    via the established CDC path — so match end drives ratings without a synchronous call and without
    a second writer to the user store.
- **Preserve the rules:** bot-vs-bot affects nobody; external games stay unrated (no rating event);
  exactly one rating mutation per human per finished match (idempotent on the match id + participant).
- **Retire** `MatchService.RecordMatchResultsAsync`'s gRPC call once the event path is proven (the
  `RecordMatchResult` RPC itself is removed in `09`).

## Contract changes
- If a new event payload is needed on `match.events`/`user.events`, add it to the proto in `01`'s
  schemas and follow the publish handoff. No new synchronous RPC.

## Tests
- A finished human-vs-human match → exactly one rating update per human (correct W/L/D + Glicko-2
  delta); replay/redelivery is idempotent (no double-count).
- Bot-vs-bot and external matches → no rating event.
- Producer/consumer glue excluded; the rating-trigger + idempotency logic covered to 100%.
- `dotnet test -p:CollectCoverage=true`; `dotnet stryker` on the trigger/idempotency logic.

## Verify
- Play a rated game to completion → both players' ratings/W-L-D update once, via events (no
  `RecordMatchResult` gRPC on the path); a replayed `MatchEnded` does not double-apply.

## Docs to update
- `rating-glicko2` / `match-history-and-stats` — note ratings are driven by `MatchEnded` events; the
  synchronous `RecordMatchResult` call is gone.
