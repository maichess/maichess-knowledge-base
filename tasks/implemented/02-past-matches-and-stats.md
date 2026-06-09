# 02 — Past matches view + real W/L/D stats

**Goal:** (a) Add a "Past Matches" view under the Profile tab listing every game a
user **played or started** (including bot-vs-bot games they spawned), and (b) make
the Profile wins/losses/draws counters reflect real data. This prompt also lays the
`created_by` + `external` match plumbing that prompts `04`/`06` reuse.

> Read `conventions.md` first.

## Background to read first

- match-manager contract: `protos/match-manager-service/v1/matches.proto`,
  `rest/match-manager.md`. Note existing `Match`, `ListMatches`/`MatchStatusFilter`
  (ongoing-only today), `MatchEvent`, and the analysis-navigation endpoints.
- match-manager service:
  `services/maichess-match-manager-service/MaichessMatchManagerService/`
  (`Entities/MatchDocument.cs`, `Services/MatchService.cs`, `Grpc/MatchesGrpcService.cs`,
  `Rest/MatchesEndpoints.cs`, `Rest/MatchListResponse.cs`/`MatchSummaryResponse.cs`,
  `Data/MatchRepository.cs`). See the Match Manager memory: Mongo collection
  `matches`, `FenHistory`, `LastMoveAt`, fire-and-forget bot-move triggers.
- user-service: `protos/user-service/v1/users.proto`, `UsersService.cs` (W/L/D are
  currently static defaults — created at 0, never updated).
- Client: `app/profile/page.tsx` (renders `user.wins/losses/draws`),
  `lib/hooks/useUserMatches.ts`, `lib/components/analysis/UserMatchList.tsx`.
  **Important:** those two are the *Analysis import* list (source:
  `app/api/analysis/matches` → analysis-service). Do **not** repurpose them; build
  the Profile Past Matches view as its own component sourced from match-manager so
  the two stay independent.

## Contract changes

### match-manager (`matches.proto` + `rest/match-manager.md`)
- Add to `Match`:
  - `Player created_by = 10;` — who initiated the match (the human who started a
    bot-vs-bot game, or the human participant for normal matches).
  - `MatchSource source = 11;` with `enum MatchSource { MATCH_SOURCE_UNSPECIFIED = 0;
    MATCH_SOURCE_NATIVE = 1; MATCH_SOURCE_EXTERNAL = 2; }` and an optional
    `string external_provider = 12;` (e.g. `lichess`, `tournament-server`; empty for
    native). `06` populates these; here they default to NATIVE.
  - `int64 finished_at_ms = 13;` if not already derivable, to support history sort.
- Add `MATCH_STATUS_FILTER_ENDED = 2;` to `MatchStatusFilter`.
- Add an RPC `ListUserMatches(ListUserMatchesRequest) returns (ListUserMatchesResponse)`
  returning matches where the user is white, black, **or** `created_by`, filtered to
  ended by default, newest first, paginated (mirror `ListMatchesRequest` paging).
- REST: document `GET /users/{user_id}/matches` (or `GET /matches?user_id=...&status=ended`)
  on match-manager returning the paginated summary list, including `source`,
  `external_provider`, and player labels.

### user-service (`users.proto` + `rest/users.md`)
- Add `RecordMatchResult(RecordMatchResultRequest) returns (RecordMatchResultResponse)`:
  inputs a `user_id` and an outcome (`enum MatchOutcome { WIN; LOSS; DRAW; }`).
  It increments the matching W/L/D counter. (Elo recompute is layered on in `03`;
  keep this RPC the single entry point so `03` extends it without a new contract.)
- Keep `elo/wins/losses/draws` in `User`.

### Versioning handoff
Per `00`: prompt the user to publish a new tagged contracts version, then bump
`Maichess.PlatformProtos` in match-manager **and** user-service (and reconcile all
other services to the same version).

## Service changes

### match-manager
- `MatchDocument` + `MatchRepository`: persist `created_by`, `source`,
  `external_provider`, `finished_at_ms`. `CreateMatch` stamps `created_by` (passed
  through from the caller — extend `CreateMatchRequest` if needed, or derive from the
  human side) and `source = NATIVE`.
- Implement `ListUserMatches` in `MatchService` (queryable; `MatchRepository` query
  stays `[ExcludeFromCodeCoverage]` per its live-Mongo exclusion, but the service-level
  filtering/mapping is fully tested).
- On match end (the existing end-of-match path in `MatchService`), call user-service
  `RecordMatchResult` for each **human** participant with their outcome. Bot-vs-bot
  games have no human participant, so `RecordMatchResult` is called for nobody — they
  **never affect any player's W/L/D or (after `03`) Elo**. They are still listed in
  Past Matches via `created_by` attribution to the user who started them. Wire the
  user-service gRPC client as a singleton like the other clients.
- Tests: extend the Reqnroll/unit suite — created_by stamping, ListUserMatches
  filtering (participant vs creator vs neither, ended-only, paging), and that match
  end calls RecordMatchResult with the correct outcome per side (win/loss/draw,
  white vs black, both-human vs human-vs-bot vs bot-vs-bot). 100% coverage.

### user-service
- Implement `RecordMatchResult` in `UsersService.cs`: read the user record, increment
  the right counter, persist. Handle unknown user / invalid id like existing RPCs.
- Tests: each outcome increments the right field; idempotency/validation paths. 100%.

## Client changes

- `lib/models/match.ts`: add `source` / `external_provider` to the match summary type.
- New `app/api/users/me/matches/route.ts` (or `app/api/matches/history`) proxy →
  match-manager `GET /users/{id}/matches`.
- New hook `lib/hooks/useMatchHistory.ts` + component `lib/components/MatchHistory.tsx`
  (pattern after `useUserMatches`/`UserMatchList` but pointed at the new endpoint;
  show opponent, result, time format, date, and a tag when `source === external`).
- `app/profile/page.tsx`: render the new Past Matches list below the stats card; the
  stats card already reads `user.wins/losses/draws`, which now carry real data.

## Verification

- match-manager + user-service: `dotnet test -p:CollectCoverage=true` → 100%;
  Stryker healthy.
- Manual: finish a human-vs-bot game → Profile W/L/D increments and the game appears
  in Past Matches; start a bot-vs-bot game → it appears under "started" with no stat
  change; ended-only filter and paging work.
- `cd maichess-client && npm run lint && npm run build`.

## Knowledge base

Record: match-manager owns match history (`created_by` attribution, `source`
native/external); match end fans out to user-service `RecordMatchResult`, which is
the single mutation point for player stats (and, after `03`, ratings).
