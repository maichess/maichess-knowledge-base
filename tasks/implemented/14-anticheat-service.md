# 14 — Anti-Cheat Service (two iterations)

> Read `conventions.md` and
> [anticheat-service.md](../../knowledge/services/anticheat-service.md) first.
> **New repo + new db instance + new infra** (see "Pre-create"). Best run after the read-model
> stages so the `flagged` field can ride the existing user replica/KTable; the matchmaking toggle
> (`11`) and Dev gate (`01`) are reused.

## Pre-create (ask the user before starting)

- **New repo `maichess-anticheat-service`** (ASP.NET).
- **`anticheat-db`** — a DatabaseService **instance** (Mongo) for cases/evidence/audit.
- Topic **`cheat.events.v1`** (keyed by userId) provisioned in topic-init.

## Goal

A service that analyses players' games, **flags** likely cheaters, propagates the flag so
matchmaking can exclude flagged players (per a user toggle), and lets devs clear false positives.

Detection is **combined**: engine correlation (primary) + statistical features. Build in **two
iterations**.

## Cross-cutting rules

- **Storage references, never duplicates, match-db.** `anticheat-db` stores cases, scores,
  evidence pointers (`match_id` + ply), and audit/unflag history. Read game data from match-db by
  id on demand — do **not** copy moves/FENs/game docs. (Fallback only if non-duplication proves
  impossible: an `anticheat_cases` collection in match-db, still referencing not copying.)
- **Pre-moves are legitimate.** The client supports pre-moves (committed before the opponent
  moves) → near-zero think time that is **not** suspicious. Read the pre-move signal from the move
  events and **exclude pre-moved plies from timing analysis** (and weight them in correlation).
  Getting this wrong is the main false-positive source — test it explicitly.
- **Flag propagation via `cheat.events.v1`.** Only the boolean flag propagates; the user read
  models (Redis replica + Match Maker KTable, from `11`) consume it and set `flagged`. Evidence
  stays in `anticheat-db`.
- Persist via the `anticheat-db` DatabaseService instance (project convention) — no direct Mongo
  driver.

## Iteration 1 — Post-game analysis

1. **Contracts first.** Add `cheat.events.v1` schema (api-contracts `events/`) and the service's
   REST/proto (dev overview + unflag; any internal RPC). Publish/bump per project policy.
2. Consume `MatchEnded` (`match.events`); fetch the finished game by id (match-db) and analyse
   **asynchronously** (rate-limited — this drives Engine load):
   - **Engine correlation:** per ply, `AnalyzePosition`/best-move → centipawn loss + top-move
     match; aggregate.
   - **Statistical:** move-time variance, accuracy spikes, rating-vs-performance — pre-move-aware.
   - Combine into a suspicion score; flag when it crosses threshold over enough games.
3. On a flag change, write the case/evidence/audit to `anticheat-db` and emit `cheat.events.v1`.
4. **Matchmaking toggle:** add the per-search toggle (allow/disallow being matched with
   previously-flagged players; default **disallow**) to the match-finding page and wire Match Maker
   to read each candidate's `flagged` from its read model and exclude accordingly. It's a
   matchmaking *filter*, not a ban.
5. **Dev unflag:** Dev-tab overview of flagged players + their evidence summary + a remove-flag
   action (dev-gated via `dev_mode`). Unflag writes an audit entry and emits a clearing
   `cheat.events` so the read models drop the flag.

## Iteration 2 — Live in-game flagging

1. A **stream processor** over the move events (`match.events` / `MoveApplied`) scores
   **incrementally during play** — maintain per-`(match, player)` running state; raise an early
   advisory suspicion signal mid-game.
2. Same **pre-move handling**. Live scoring is **advisory**; the authoritative post-game pass
   (iteration 1) reconciles it.
3. Decide and document how a live signal surfaces (e.g. an internal event / dev visibility) versus
   the durable flag, which still comes from the reconciled verdict.

## Deployment (required)

- Helm: `maichess-anticheat-service` Deployment + `anticheat-db` DatabaseService instance (Mongo)
  + `cheat.events.v1` topic. Engine must be reachable; document expected (async, throttled) Engine
  load.
- Client: matchmaking toggle + Dev overview/unflag; `npm run build && npm run lint`.

## Tests (mandatory)

- Detector: correlation scoring, statistical scoring, **pre-move exclusion** (a fast pre-moved
  ply must not raise timing suspicion), threshold/flag logic.
- Flag lifecycle: flag → `cheat.events` → read-model `flagged` set; unflag → cleared; audit
  written.
- Matchmaking: flagged players excluded when disallowed, included when allowed; flagged searcher
  handling.
- Iteration 2: incremental scoring on a move stream; reconciliation with the post-game verdict.
- 100% on non-excluded code; Stryker wired to mirror exclusions; run with coverage.

## Verify

1. Play an engine-assisted game in a test env → player gets flagged; evidence visible in Dev.
2. With "disallow flagged" on, a flagged player is not matched to you; toggle off → allowed.
3. Dev removes the flag → player is matchable again; audit recorded.
4. A heavy pre-mover is **not** flagged on timing alone.

## Checklist

- [ ] Repo + `anticheat-db` + `cheat.events.v1` pre-created.
- [ ] Contracts published (publish/bump) before code.
- [ ] Iter 1: post-game combined detector; cases reference (not copy) match-db; flag via events.
- [ ] Matchmaking toggle + Dev unflag (dev-gated).
- [ ] Iter 2: live stream-processor scoring; pre-move-aware; advisory + reconciled.
- [ ] Helm: service + db instance + topic.
- [ ] Tests to 100% incl. pre-move exclusion; Stryker wired.
- [ ] Decisions recorded against [anticheat-service.md](../../knowledge/services/anticheat-service.md).
