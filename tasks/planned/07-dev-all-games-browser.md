# 07 — Dev "All games" browser

**Goal:** Add a Dev-section tab that lists **every game played on maichess**, sorted
**chronologically**, with **filter options** by **player** (a participant) and by
**initiator** (the `created_by` user, for bot-vs-bot games). This is a global
browser across native human games, arena bot-vs-bot games, and mirrored external
games — all surfaced uniformly with their source tag.

> Read `conventions.md` first.
> **Depends only on `01`** (the `dev_mode` gate + `/dev` shell/nav) **and `02`** (the
> `Match.created_by` / `Match.source` fields and match-manager match-listing infra).
> It is independent of `03`–`06` and can be implemented any time after `02`.

## Background to read first

- match-manager: `02` added `created_by`, `source` (`NATIVE`/`EXTERNAL`),
  `external_provider`, `finished_at_ms`, the `MATCH_STATUS_FILTER_ENDED` value, and
  `ListUserMatches`. This prompt adds a **global, filterable** query alongside those —
  reuse the same `MatchDocument` fields, summary mapping, and paging conventions.
  (`Services/MatchService.cs`, `Data/MatchRepository.cs`, `Rest/MatchesEndpoints.cs`,
  `Rest/MatchSummaryResponse.cs`/`MatchListResponse.cs`.)
- Client Dev section: `01` left the `dev_mode`-gated `/dev` shell + nav entry; `05`
  adds sibling Dev tabs (Spawn / Results). Add "All games" as another Dev tab using the
  same `01` guard helper and the proxy pattern (`lib/utils/proxy.ts`,
  `app/api/.../route.ts`).
- Label sources: `lib/hooks/useBots.ts` for bot names; usernames come from the match
  summary / user-service.

## Contract changes (`maichess-api-contracts`)

### match-manager (`matches.proto` + `rest/match-manager.md`)
- Add `SearchMatches(SearchMatchesRequest) returns (SearchMatchesResponse)`:
  - `string player_id` — match where this user is **white or black** (empty = no
    participant filter).
  - `string initiator_id` — match where `created_by` is this user (empty = no filter);
    primarily meaningful for bot-vs-bot games.
  - `MatchStatusFilter status` — reuse the existing enum (`ONGOING` / `ENDED` /
    `UNSPECIFIED` = all).
  - Optional `MatchSource source` filter (all / native / external).
  - Optional `int64 since_ms` / `int64 until_ms` time-range bounds.
  - `bool ascending` — chronological sort direction (default newest-first).
  - Paging (`page`, `page_size`) mirroring `ListMatchesRequest`.
  - Response: paginated match summaries including `white`, `black`, `created_by`,
    `status`, `source`, `external_provider`, time format, move count, and the
    chronological timestamp.
- REST: `GET /matches/search` (or extend `GET /matches`) with the above as query
  params, returning the paginated summary list.
- **Access model:** the endpoint requires a valid bearer token. This does **not**
  widen data exposure — ended matches are already publicly viewable via
  `GET /matches/{id}` and ongoing ones are already listed by Watch — so a global
  chronological list is consistent with the existing model. The **UI** is
  `dev_mode`-gated; if you additionally want server-side dev gating, enforce it at the
  client proxy (the proxy already has the user) rather than coupling match-manager to
  user-service.
- Versioning handoff per `00`: publish a new tagged contracts version; bump
  `Maichess.PlatformProtos` across all services.

## Service changes (`maichess-match-manager-service`)

- Implement `SearchMatches` in `MatchService`: build the query from the filters and map
  to summaries, reusing the `ListUserMatches` mapping/paging code from `02` (extract a
  shared helper rather than duplicating). The `MatchRepository` query method stays
  `[ExcludeFromCodeCoverage]` (live Mongo), but the **filter-building and summary
  mapping in the service layer are fully tested**.
- Wire the gRPC method in `MatchesGrpcService` and the REST route in `MatchesEndpoints`
  (REST adapter stays excluded).
- Tests (100% on non-excluded): no-filter returns all (paged, newest-first);
  `player_id` filters to participant games; `initiator_id` filters to created-by games;
  `status` and `source` filters; `ascending` flips order; `since_ms`/`until_ms` bounds;
  combined filters; empty result. Update the Stryker config for any new files.

## Client changes (`maichess-client`)

- Proxy `app/api/dev/games/route.ts` → match-manager `GET /matches/search`, forwarding
  the filter query params with bearer auth (`getBearerToken`).
- `lib/models/match.ts`: add the search-summary type if not already present from `02`.
- Hook `lib/hooks/useAllGames.ts`: holds filter state (`playerId`, `initiatorId`,
  `status`, `source`, `ascending`), pagination, and (optionally) light polling for the
  live view; debounce text filters.
- Page `app/dev/games/page.tsx` (guarded by the `01` `dev_mode` helper):
  - **Filter controls:** a **player** filter (by username/id) and an **initiator**
    filter (by username/id, for bot games), plus status/source toggles and a
    sort-direction toggle.
  - **Chronological list:** columns for timestamp, White, Black, result, a source tag
    (Native / External provider / arena bot-vs-bot), and the **initiator** column shown
    for bot games. Each row links into the existing match/Watch viewer.
  - Empty/loading/error states matching the existing list components.
- Add the "All games" entry to the Dev landing / nav (coordinate with `05`'s Dev tabs;
  if `05` hasn't run yet, this prompt adds the tab and `05`'s landing links to it).

## Verification

- `cd services/maichess-match-manager-service && dotnet test -p:CollectCoverage=true`
  → 100% on non-excluded; `dotnet stryker` → review surviving mutants in filter-building.
- `cd maichess-client && npm run lint && npm run build`.
- Manual (Dev mode on): open "All games" → all games appear newest-first; filtering by
  a player narrows to their games; filtering by an initiator narrows to bot games that
  user started; flipping sort order works; rows open the match viewer; external games
  show the external tag.

## Knowledge base

Note in `maichess-knowledge-base/` that the Dev "All games" browser is backed by
match-manager `SearchMatches` (global, filter by participant + initiator, chronological)
and is `dev_mode`-gated at the UI, consistent with the existing public-match access
model.
