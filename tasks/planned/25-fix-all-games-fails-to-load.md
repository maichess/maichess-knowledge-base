# 25 — Fix: "All games" browser fails to load

> Read `conventions.md` and
> [match-history-and-stats](../../knowledge/domain/match-history-and-stats.md) first.
> Closely tied to task **07** (Dev all-games browser, 🟡 in progress). Touches
> match-manager (+ contracts) and possibly the client proxy.

## Symptom (item 10)

The Tools → **All games** view fails to load games (shows "Failed to load games.").

## What's wired today

- Client: `lib/hooks/useAllGames.ts` → `GET /api/dev/games?...`.
- Proxy: `app/api/dev/games/route.ts` forwards to match-manager
  **`GET /matches/search`** with bearer auth and returns the upstream status verbatim.
- So a non-2xx (or shape-mismatched) response from match-manager `/matches/search`
  surfaces as the load failure.

## Most likely root cause

Per task **07**'s status (memory + ROADMAP: 🟡, *"blocked on v0.9.0 contracts publish +
.NET verify"*), the `SearchMatches` / `GET /matches/search` endpoint that this view
depends on is **not fully shipped/deployed**: either the contract version was never
published+tagged, the match-manager build wasn't bumped/verified against it, or the
endpoint isn't deployed in the running environment. That makes the proxy call 404/500.

## Steps

1. **Reproduce + capture the upstream status.** Hit `/matches/search` directly against
   the running match-manager (with a valid bearer) and record the status/body. Add a
   transient log of the upstream status in the proxy if needed. This tells you whether
   it's "endpoint missing" (404), "auth" (401/403), or "shape/serialization" (500/200
   with wrong shape).
2. **If endpoint missing/contract-blocked:** finish task 07's handoff —
   publish + tag the contracts version, bump `Maichess.PlatformProtos` in match-manager
   (and the client if it consumes the TS package), redeploy, and **the user runs the
   .NET verify** (per `feedback_contract_restore_creds`: Claude's shell can't restore the
   freshly published package). Document any blocker in match-manager `CONTRACT_NOTES.md`.
3. **If shape mismatch:** align the match-manager response to the `MatchListResponse`
   shape the client expects (`{ matches, total, page, page_size }`) and the query params
   the proxy forwards (`player_id, initiator_id, status, source, since_ms, until_ms,
   ascending, page, page_size`).
4. Mark task 07 `✅` once the browser loads end-to-end; otherwise keep it `🟡` and update
   its spec with the precise remaining blocker.

## Testing

- Match-manager: unit tests for `SearchMatches` filter/paging branches (100% on new code).
- Client: manual load of Tools → All games with and without filters; confirm rows render
  and paginate.
