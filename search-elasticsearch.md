# Search (Elasticsearch)

**Status:** Accepted — implementation in `feature-prompts/13`
**Relates to:** [change-data-capture.md](change-data-capture.md),
[caching-and-read-models.md](caching-and-read-models.md),
[analysis-service.md](analysis-service.md)

## Context

The analysis games library grows forever and is never deleted; Past Matches will grow large per
user. Mongo equality filters (the generic DatabaseService contract) cannot do full-text or rich
faceted search, and "find games that reached this position" is a genuinely valuable chess-product
query that no key lookup can serve. These are search problems, not cache problems.

## Decision

Adopt **Elasticsearch as a derived read model** for search, owned by a **new
`maichess-search-service`**. Full initial scope:

1. **Games library search** — index `analysis_games`: opponent name, opening / ECO, result,
   date, tags, full-text over PGN headers.
2. **Match-history facets** — index `matches`: filter Past Matches by opponent, source/provider,
   result, date range.
3. **Position search** — "find games/matches that reached this FEN." Index a **normalised
   placement key** (the piece-placement field of the FEN, side-to-move optionally folded in) per
   ply so a position lookup is an exact-term query, not a scan.

### Hard constraints

- **Elasticsearch is never a system of record.** Mongo remains authoritative. The index is
  rebuildable at any time from the source collections (replay CDC / reindex job). If the cluster
  is lost, nothing is lost.
- **Fed from Kafka/CDC, not dual-written.** The indexer in `maichess-search-service` consumes
  `match.cdc.v1` (Mongo Debezium) and the analysis-game change stream and projects into ES. The
  owning services (analysis, match-manager) do **not** call Elasticsearch.
- **Direct ES access is a documented exception** to the "persist via DatabaseService" rule, on
  the same grounds as the Redis read model: ES is a derived search/read engine, not the domain's
  master store. `maichess-search-service` talks to ES directly via the official client.

### Position-search indexing

- Per game, emit one indexed entry per ply: `{ game_id|match_id, ply, placement_key, fen }`.
- `placement_key` = FEN field 1 (piece placement), normalised (no move counters), optionally
  with side-to-move. Exact-term match → all games/plies reaching the position.
- Keep this in a dedicated `positions` index (or nested docs) so library/match indexes stay lean.

## API

`maichess-search-service` exposes a REST API (auth via the shared JWT, same JwtBearer pattern):

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/search/games` | faceted/full-text over the user's analysis games |
| `GET` | `/search/matches` | faceted Past Matches search for the authenticated user |
| `GET` | `/search/positions?fen=` | games/matches that reached a position |

Results return ids + summary fields; clients hydrate detail from the owning service as today.
The REST contract is specified in `maichess-api-contracts/rest/search.md` before implementation.

## Deployment

- New Helm components: an **Elasticsearch** StatefulSet (single node in staging, 3-node in prod)
  + the `maichess-search-service` Deployment.
- The Mongo Debezium connector from [change-data-capture.md](change-data-capture.md) is the feed;
  add a one-shot **reindex job** that backfills ES from the current collections on first rollout
  and on demand.
- Index templates / mappings ship as part of the service's startup or an init job, versioned in
  `maichess-search-service`.

## Non-goals

- Leaderboards are **not** ES — they are Redis sorted sets
  (see [caching-and-read-models.md](caching-and-read-models.md)). Do not reach for ES for ranked
  numeric lookups.
