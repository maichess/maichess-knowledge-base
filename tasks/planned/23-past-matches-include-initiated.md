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
