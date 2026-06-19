# 06 — insights query API

> Read [conventions.md](../../conventions.md),
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md), and
> [caching-and-read-models](../../../knowledge/architecture/caching-and-read-models.md) first.
> `maichess-insights-service` (.NET). Depends on jobs writing `insights_*` (04) and the control plane (05).

## Goal

Serve the computed insights over REST + gRPC, reading the materialized `insights_*` collections, with a
rebuildable Redis L1 cache on the hot aggregates.

## What already exists (reuse it)

- The `insights_*` collections written by task 04 and the metric shapes in
  [insights-statistics](../../../knowledge/domain/insights-statistics.md).
- DatabaseService gRPC read pattern (`Database.DatabaseClient` + `Struct` decoding) — read `insights_*`
  from `insights-db` exactly as analysis-service reads its store; **no direct Mongo driver**.
- The analysis-service **L1 Redis cache seam** (`IInsightsCache` analogue): `allkeys-lru`-tolerant,
  rebuildable from Mongo on miss.

## Implementation

1. **`Services/InsightsQueryService`** (fully covered): for each query map a `(corpus_id, paging,
   filters)` request → repository read → response DTO. Logic (paging, filter validation, cache
   key-building, miss→read→populate) is unit-tested.
2. **`Data/InsightsRepository`** (`[ExcludeFromCodeCoverage]`): the database-service gRPC read calls
   behind an `IInsightsRepository` seam.
3. **`IInsightsCache`** Redis L1 over the hot aggregates (top openings, summary, tricky), keyed by
   corpus id + query params; rebuildable.
4. **Endpoints** (REST + gRPC, implementing task 01): `GetTopOpenings`, `GetCommonEndgames`,
   `GetCommonPositions`, `GetTrickyPositions`, `GetCorpusSummary`, `ListCorpora` — paged, corpus-scoped.

## Contract changes

None new (implements task 01). Any gap → publish handoff + `CONTRACT_NOTES.md`.

## Tests (mandatory)

- `dotnet test -p:CollectCoverage=true` — **100%** non-excluded. Reqnroll/unit: query→repository
  mapping, paging/filter edge cases, cache miss→populate and hit paths (fake repository + fake cache).
  Stryker mirrored. Excluded: repository, cache client glue, REST handlers, DTO records, `Program.cs`.

## Verify

- On staging, after task 04 has populated a corpus: the query endpoints return correct top openings /
  endgame signatures / common & tricky positions / summary; a second call is served from Redis; flushing
  Redis and re-querying rebuilds from Mongo.

## Knowledge base

- Note the insights L1 cache in
  [caching-and-read-models](../../../knowledge/architecture/caching-and-read-models.md). Mark task 06
  `🟡` in [ROADMAP.md](../../ROADMAP.md).
