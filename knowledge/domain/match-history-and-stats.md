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
