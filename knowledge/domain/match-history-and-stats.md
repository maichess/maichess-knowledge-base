# Match history & player stats

## Decision

**Match Manager owns match history.** Every match carries:

- `created_by` — the initiator. The human participant for normal matches, or
  the human who started a bot-vs-bot game. When `CreateMatch` is called without
  it, Match Manager derives it from the human side (white if human, else black
  if human, else nobody for bot-vs-bot). This attribution is what makes a
  bot-vs-bot game appear in its starter's Past Matches even though they occupy
  neither colour.
- `source` (`native` | `external`) + `external_provider` — native for all games
  played on the platform; `external` is reserved for games mirrored in from
  another provider (Lichess, tournament-server) by the external-games feature.
  Until then everything is `native`.
- `finished_at_ms` — stamped when the match transitions to an ended status;
  used to sort Past Matches newest-first.

**Past Matches** is served by `ListUserMatches` (gRPC) / `GET /users/{user_id}/matches`
(REST): matches where the user is white, black, **or** `created_by`, filtered to
ended by default, newest first, paged. Membership/status filtering, ordering and
paging are done in the service layer (`MatchService.ListUserMatchesAsync`); the
repository fetch (`FindForUserAsync`) is the only DB-touching part and stays
`[ExcludeFromCodeCoverage]`. This list is **independent** of the Analysis import
list (`analysis-service GET /matches` → client `useUserMatches`/`UserMatchList`);
the Profile Past Matches view uses its own `useMatchHistory`/`MatchHistory`
sourced from Match Manager.

## Dev "All games" browser

A global, cross-user view (the Profile Past Matches above is per-user) lives under
the Dev section: **every** game on the platform — native human, arena bot-vs-bot,
and mirrored external — in one chronological feed, filterable by **participant**
(`player_id` = white or black), **initiator** (`initiator_id` = `created_by`,
primarily for bot-vs-bot games), status, source, and time range, with a sort
direction toggle. It is served by `SearchMatches` (gRPC) / `GET /matches/search`
(REST) on Match Manager.

- **Same shape as Past Matches, generalised.** `MatchService.SearchMatchesAsync`
  reuses the extracted `OrderAndPage` helper (shared with `ListUserMatchesAsync`)
  and applies membership/status/source/time filtering in the service layer; the
  candidate-set fetch (`MatchRepository.SearchAsync`) is the only DB-touching part
  and stays `[ExcludeFromCodeCoverage]`. `player_id` and `initiator_id` are ANDed;
  the chronological order and the `since_ms`/`until_ms` bounds key on
  `finished_at_ms`.
- **Whole-collection candidate fetch (no-scope path).** With no `player_id`/
  `initiator_id` scope — the default browse — `SearchAsync` lists the **entire**
  `matches` collection in one `Database.List`, then the service filters/orders/pages
  it in memory. That single `ListResponse` can be large, so the match-manager →
  match-db gRPC client is built with `MaxReceiveMessageSize = null` (Program.cs);
  the stock 4 MB default would `RESOURCE_EXHAUSTED` past a few hundred matches and
  surface as "Failed to load games" (task 25). The scoped path runs cheap equality
  lookups (white/black/created_by) and is naturally small. This fetch-all-then-
  filter design is intentional for a low-volume Dev tool; the scale lever, if ever
  needed, is DB-side ordering + paging (`ListRequest` already has `limit`/`offset`;
  a `Count` RPC already exists; only a `sort` field would be missing).
- **Read, so synchronous (post-Kafka).** Like the other list reads it queries the
  durable match-db materialised by the projector; it deliberately does **not**
  overlay the Redis live read model — rows link into the match/Watch viewer, which
  performs the live overlay. Consistent with the CQRS read/write split.
- **Access model.** A valid bearer token suffices: ended matches are already
  readable via `GET /matches/{id}` and ongoing ones are listed by Watch, so a
  global chronological list widens nothing. The **UI** is `dev_mode`-gated
  (`requireDevUser` + the `/api/dev/games` proxy as the server-side gate); Match
  Manager is not coupled to user-service for this.
- **Relationship to [[search-service]].** Complementary, not duplicative.
  `maichess-search-service GET /search/matches` is **per-user** faceted/full-text/
  position search over the Elasticsearch read model with best-effort id labels;
  `SearchMatches` is a **cross-user** chronological feed with Match-Manager-resolved
  player labels (usernames + bot names) and first-class initiator attribution. The
  Dev UI cross-links the two ("All games" ↔ "Search"); use Search for full-text /
  opening / position lookups, the All-games browser for structured chronological
  browsing.
- **Client** (`maichess-client`): `app/api/dev/games` proxy → `/matches/search`;
  `lib/hooks/useAllGames.ts` (filter + pagination state, debounced text filters,
  optional light polling for the live view); `lib/components/dev/AllGamesPanel.tsx`
  (filters + chronological table with a source tag and an initiator column);
  `app/tools/games/page.tsx` (under the **Tools** tab, guarded by `requireUser` —
  moved out of `/dev` in the 2026-06-12 UX batch; the `/api/dev/games` proxy path is
  unchanged). The Tools landing links it alongside Bot arena and Search.

Contract introduced in `Maichess.PlatformProtos` v0.9.0: `SearchMatches` RPC +
`SearchMatchesRequest`/`SearchMatchesResponse` in
`protos/match-manager-service/v1/matches.proto`; `GET /matches/search` in
`rest/match-manager.md`.

## Player stats: single mutation point

When a match ends (via a move, resignation, draw acceptance, or timeout
enforcement) the projector's **`MatchEnded`** on `match.events.v1` — enriched
with both participants, the source, and the bot sides' creation-time elo —
drives the stats update: user-service's `MatchEndedConsumer` applies
`UsersService.ApplyMatchEndedAsync` once per **human** participant with their
`WIN`/`LOSS`/`DRAW` outcome (kafka task 08; the synchronous
`RecordMatchResult` gRPC call is gone from this path, the RPC itself is removed
in kafka 09). This is the **single entry point** for player-stat mutations:

- Bot-vs-bot games have no human participant, so they record nothing and never
  affect any player's W/L/D or rating. They still appear in Past Matches via
  `created_by`.
- External matches never enter the event loop (no `MatchEnded` fact), and the
  consumer additionally skips `source = external` — external games stay unrated.
- Exactly one mutation per human per match: the `rated_matches` marker on the
  users row commits in the same row update as the stats it guards, so a
  redelivered or replayed `MatchEnded` is a no-op.

`ApplyMatchEndedAsync` reads the user record, increments the matching counter,
recomputes the Glicko-2 rating (see [[rating-glicko2]]), and persists via the
generic database-service CRUD (no direct SQL).

## Where it lives

- **Contract** (introduced in `Maichess.PlatformProtos` v0.3.4):
  - `Match.created_by` (10), `source` (11) + `MatchSource` enum,
    `external_provider` (12), `finished_at_ms` (13);
    `CreateMatchRequest.created_by` (4); `MatchStatusFilter.MATCH_STATUS_FILTER_ENDED`;
    `ListUserMatches` RPC in `protos/match-manager-service/v1/matches.proto`.
  - `RecordMatchResult` RPC + `MatchOutcome` enum in
    `protos/user-service/v1/users.proto` (internal gRPC, no public REST endpoint).
  - REST: `GET /users/{user_id}/matches` and the extended match summary
    (`created_by`, `source`, `external_provider`, `finished_at_ms`) in
    `rest/match-manager.md`; the stat-mutation note in `rest/users.md`.
- **Client** (`maichess-client`): `MatchSource` + the new summary fields in
  `lib/models/match.ts`; `app/api/users/me/matches` proxy (resolves the
  authenticated user, then calls match-manager); `lib/hooks/useMatchHistory.ts`
  + `lib/components/MatchHistory.tsx`; Past Matches section in
  `app/profile/page.tsx`. The stats card already reads `user.wins/losses/draws`,
  which now carry real data.
