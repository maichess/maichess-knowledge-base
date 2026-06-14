# 23 — Past matches: include the user's just-initiated (in-progress) games

> Read `conventions.md` and
> [match-history-and-stats](../../knowledge/domain/match-history-and-stats.md) first.
> Touches match-manager listing + `maichess-client` (`UserMatchList` / `useUserMatches`).

## Goal

The Analysis → **Past matches** tab currently lists only *finished* matches the user
played. Also surface games the user has **just initiated** (still in progress) so they
can jump back into review/analysis without waiting for the game to end.

## Background

- Client: `lib/components/analysis/UserMatchList.tsx` + `lib/hooks/useUserMatches.ts`.
  The "Analyse" action imports the match (`POST /api/games/from-match/:id`) then opens
  the analysis view.
- The list is backed by match-manager's per-user match listing. Today it filters to
  finished statuses. Confirm the exact endpoint/params in
  `maichess-api-contracts/rest/match-manager.md` (and the proto) before changing —
  do not infer from the client.

## Decisions to make

- **Which "initiated" games?** Games where the user is `created_by` and/or a player and
  status is `ongoing`. Recommended: include ongoing games where the user is **a player**
  (consistent with the dashboard "Continue playing" fix). If product wants
  spawned-but-not-played games too, gate that separately.
- **Importing an in-progress game:** `from-match` import must tolerate an unfinished
  match (partial move list). Verify the analysis-service import path handles ongoing
  matches; if it rejects them, that's the blocker to resolve (document in
  `CONTRACT_NOTES.md` if a contract change is needed).

## Implementation sketch

- Match-manager: allow the user-matches listing to include `ongoing` (e.g. a
  `status=all|ongoing|finished` param or an `include_ongoing` flag), defaulting to
  current behavior so other callers are unaffected.
- Client: `useUserMatches` requests the broader set for the Past-matches tab; render an
  "in progress" badge on ongoing rows; the row action opens analysis (read-only by
  default — see task 22).

## Testing

- Match-manager: unit tests for the listing with/without ongoing games (100% on new code).
- Client: `npm run build` + `npm run lint` + manual check (start a game, see it appear
  as in-progress in Past matches, open it).

---

## Resolution (shipped 2026-06-14)

**The backing service was analysis-service, not match-manager.** The Analysis → Past matches
tab (`UserMatchList`/`useUserMatches`) is served by analysis-service `GET /matches` (reads
match-db directly), **not** Match Manager's `ListUserMatches` (which powers the *Profile* Past
Matches view via `useMatchHistory`/`MatchHistory`). Confirming the endpoint in the contract
before coding — as the spec instructed — surfaced this; match-manager was left untouched.

Contract (REST-only, no proto change → **no `Maichess.PlatformProtos` version bump**):
- `GET /matches` gained a `status` query param (`finished` default | `ongoing` | `all`).
- `POST /games/from-match/{match_id}` now accepts ongoing matches (imports a point-in-time
  snapshot, result `*`) instead of rejecting them with `400`.

Implementation:
- **analysis-service** — `AnalysisGameService.ParseStatusFilter` + a `UserMatchStatusFilter`
  arg on `ListUserMatchesAsync`; the ongoing rejection (`MatchStillOngoingException`, now
  deleted) dropped from `ImportFromMatchAsync`. Tests at 100% line/branch/method.
- **client** — `useUserMatches` requests `status=all`; `UserMatchList` shows an "In progress"
  badge on ongoing rows. Opening one imports the snapshot and lands in the read-only analysis
  viewer (task 22).

See [analysis-service](../../knowledge/services/analysis-service.md) ("match-db access for the
match list and match import").
