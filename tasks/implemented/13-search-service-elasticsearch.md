# 13 — Caching Stage 5: maichess-search-service (Elasticsearch)

> Read `conventions.md`,
> [search-elasticsearch.md](../../knowledge/services/search-service.md), and
> [change-data-capture.md](../../knowledge/architecture/change-data-capture.md) first.
> **New repo + new infrastructure** (see "Pre-create" below). Depends on the Mongo Debezium
> connector — extend `10`'s Connect setup with a Mongo connector here.

## Pre-create (ask the user before starting)

- **New repo `maichess-search-service`** (ASP.NET, consistent with the other .NET services).
- **Elasticsearch** cluster (Helm).
- **Debezium Mongo connector** on `match-db` → `match.cdc.v1` (Kafka Connect from `10`).

## Goal

Elasticsearch as a **derived read model** for search, owned by `maichess-search-service`. Full
scope: games-library search, match-history facets, and FEN **position search**.

## Hard constraints (from the ADR)

- **ES is never a system of record.** Mongo stays authoritative; every index is rebuildable from
  source (reindex job / CDC replay).
- **Fed from CDC/Kafka, not dual-written.** The indexer consumes `match.cdc.v1` (and the analysis-
  game change stream) and projects into ES. analysis-service and match-manager do **not** call ES.
- **Direct ES access is a documented exception** to the persist-via-DatabaseService rule (ES is a
  derived search engine, like the Redis read model). Note this in the service `CONTRACT_NOTES.md`.

## Implementation

1. **Contracts first.** Specify the search REST API in
   `maichess-api-contracts/rest/search.md`:
   - `GET /search/games` — faceted/full-text over the user's analysis games (opponent, opening/
     ECO, result, date, tags, free text on PGN headers).
   - `GET /search/matches` — faceted Past Matches for the authenticated user.
   - `GET /search/positions?fen=` — games/matches that reached a position.
   Auth via the shared JWT (same JwtBearer + `access_token` cookie pattern as match-manager).
2. **Indexer (consumer).** Consume `match.cdc.v1` + the analysis-game changes; project into ES
   indexes:
   - `analysis_games`, `matches` — summary + facet fields.
   - `positions` — one entry per ply: `{ game_id|match_id, ply, placement_key, fen }`, where
     `placement_key` = normalised FEN piece-placement (no move counters; side-to-move optional).
3. **Search API.** Query ES; return ids + summary fields; the client hydrates detail from the
   owning service (no detail duplication).
4. **Index templates/mappings** ship in the service (startup or init job), versioned.
5. **Reindex/backfill job** that builds ES from current Mongo collections on first rollout and on
   demand (the rebuild path the ADR requires).

## Deployment (required)

- Helm: **Elasticsearch** StatefulSet (1 node staging / 3 prod) + **`maichess-search-service`**
  Deployment + the **Debezium Mongo connector** + the reindex Job.
- Wire the service's Kafka consumer, ES endpoint, and JWT key via config like the other services.
- Client: add the search UI wiring (Dev or library views) per existing client conventions; verify
  with `npm run build && npm run lint`.

## Tests (mandatory)

- Indexer: CDC change → correct ES document(s), including per-ply position entries and
  placement-key normalisation. Test against an ES test container or a mocked ES client per the
  service's chosen approach.
- Search API: faceted query mapping, position lookup by FEN, auth scoping (a user only searches
  their own games/matches where applicable).
- 100% on non-excluded code (REST adapters/ES client excludable like other infra deps); wire
  Stryker.NET to mirror exclusions; run with coverage.

## Verify

1. Import games, play matches; `GET /search/games` / `/search/matches` return correct faceted
   results; `GET /search/positions?fen=` finds games that reached the position.
2. Drop the ES cluster and run the reindex job — search fully recovers from Mongo (proves ES is
   derived, not authoritative).

## Checklist

- [ ] `maichess-search-service` repo + ES + Mongo Debezium connector pre-created.
- [ ] `rest/search.md` contract published (publish/bump handoff) before code.
- [ ] Indexer projects `analysis_games` / `matches` / `positions` from CDC.
- [ ] Position search via normalised placement_key.
- [ ] ES-as-derived documented in `CONTRACT_NOTES.md`; reindex job exists.
- [ ] Helm: ES + service + connector + reindex job.
- [ ] Tests to 100%; Stryker wired.
- [ ] Decisions recorded against [search-elasticsearch.md](../../knowledge/services/search-service.md).
