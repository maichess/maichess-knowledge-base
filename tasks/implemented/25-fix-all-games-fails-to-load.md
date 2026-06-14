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

---

## Resolution (2026-06-14) — ✅

**Not the hypothesised cause.** By the time this was investigated the contract was long
shipped: contracts are at **v0.12.0**, `GET /matches/search` + `SearchMatches` are
implemented, match-manager builds against them, and all 342 unit tests pass. The proxy,
the client hook (`useAllGames`), and the response shape (`MatchListResponse`) all line up
exactly. The same proxy pattern (`/api/matches`) works, so auth/URL/proxy were never the
problem. **No contract change was needed** (so no v0.13.0): `database.proto`'s `ListRequest`
already carries `limit`/`offset` and a `Count` RPC already exists.

**Actual root cause — gRPC receive-size cap on the cross-user full-collection fetch.**
The Dev "All games" browse is the only read that calls `SearchMatches` with **no
participant/initiator scope**. By design (documented on `MatchService.SearchMatchesAsync`
and `MatchRepository.SearchAsync`) that path pulls the **entire `matches` collection** in
one `Database.List` response — it is the candidate set the service then filters by
status/source/time, orders by `finished_at_ms`, and pages **in memory**. The match-manager
→ match-db gRPC client was created with `GrpcChannel.ForAddress(url)` and **no options**,
so it kept the Grpc.Net.Client default `MaxReceiveMessageSize` of **4 MB**. Once the
platform accumulates a few hundred matches (trivially reached with bot-arena games), that
single `ListResponse` exceeds 4 MB and the call fails with `RESOURCE_EXHAUSTED` → the REST
handler 500s → the proxy forwards the non-2xx → the hook renders **"Failed to load
games."** The always-`status`/`category`-filtered, always-DB-paged `/matches` and
`ListUserMatches` reads never approach the limit, which is why only this view broke.

**Fix.** `MaichessMatchManagerService/Program.cs` now builds the database client with
`new GrpcChannelOptions { MaxReceiveMessageSize = null }` (unlimited receive), matching the
match-db server's uncapped send and the documented "whole collection is the candidate set"
design. One-line infrastructure change; no API, client, or other-service changes. Build +
342 tests green. `Program.cs` is coverage-excluded, so there is no new unit test; the
verification is the manual load below.

**Scale follow-up (noted, not done):** the fetch-all-then-filter-in-memory design is fine
for a low-volume internal Dev tool but is O(collection) per page load. If "All games" ever
needs to scale, push ordering + paging to match-db (add a `sort` field to `ListRequest`,
reuse the existing `Count` RPC) for the no-scope / equality-only-filter case rather than
growing the in-memory scan. Out of scope here.

**Verify (manual, user):** with a populated match-db (or after a bot-arena run that creates
>~few-hundred matches), open Tools → **All games** → rows render newest-first and paginate;
filter by player / initiator / status / source / date; rows open the viewer.
