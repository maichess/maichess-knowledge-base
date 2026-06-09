# 12 — Caching Stage 4: analysis-results L1 + leaderboards (Redis ZSET)

> Read `conventions.md` and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md) first.
> Depends on `11` for the user replica/rating stream. **No new infrastructure** (reuses Redis).

## Goal

Two independent, low-risk wins:

1. A **Redis L1** in front of the existing Mongo `analysis_results` cache for the default bot's
   hot positions.
2. **Leaderboards** as Redis **sorted sets** — the right structure for ranked-by-rating queries
   (explicitly *not* Elasticsearch, not a Postgres `ORDER BY`).

## Part A — analysis_results L1

Context: `analysis_results` (see
[analysis-service.md](../../knowledge/services/analysis-service.md)) is already a durable Mongo
cache of per-depth engine analysis for the default bot.

1. Add a Redis L1 keyed by `(fen, bot_id)` holding the depth-ordered lines; on a session start,
   check L1 → Mongo L2 → engine, in that order.
2. Populate L1 on Mongo hits and on new default-bot depth writes. Keep Mongo as the durable L2.
3. Respect the existing rules: only the configured `DefaultAnalysisBotId` is cached; the startup
   bot-change scrape (`analysis_meta`) must also clear the L1, not just Mongo.
4. L1 is **rebuildable** from Mongo — a TTL or LRU eviction is fine here (unlike Stage 1, the
   underlying data is a cache already).

## Part B — leaderboards (Redis ZSET)

1. Maintain a Redis sorted set (e.g. `leaderboard:rating`) of `userId → rating`, updated by
   consuming the rating event on `user.events` (reuse the Stage 3 replica consumer; do not add a
   second source of truth).
2. Serve ranked queries with `ZREVRANGE` (top N) and `ZREVRANK` ("your rank") — O(log N), no DB
   scan.
3. Exclude/annotate as product requires (e.g. provisional ratings, or `flagged` players once
   anti-cheat lands — read `flagged` from the same replica).
4. Expose via the owning service that already serves user/profile data (or wherever the
   leaderboard UI is fed); specify the REST shape in api-contracts before implementing if a new
   endpoint is added.

## Deployment (required)

- Reuses Redis. Document the new namespaces: `analysis:l1:{bot}:{fen}` (TTL/LRU, rebuildable) and
  `leaderboard:rating` (rebuildable by replaying `user.events`).
- Note that both are drop-safe.

## Contracts

- Part A: none (internal cache).
- Part B: if a leaderboard REST endpoint is added, follow the standard api-contracts publish/bump
  handoff; otherwise none.

## Tests (mandatory)

- Part A: L1 hit/miss/promote, bot-change scrape clears L1, depth trimming preserved. Keep
  analysis-service coverage rules (it persists via DatabaseService — mock it).
- Part B: ZSET upsert on rating event, top-N and rank queries, flagged/provisional handling.
- 100% on non-excluded code; mirror Stryker exclusions; run with coverage.

## Verify

1. Repeated analysis of the same hot position serves from Redis L1 (no Mongo round-trip on the
   second request); changing the default bot clears it.
2. Leaderboard top-N and "your rank" return correctly and update after a rated game; flushing
   Redis rebuilds both from their sources.

## Checklist

- [ ] analysis_results Redis L1 over Mongo L2, default-bot only, scrape clears L1.
- [ ] `leaderboard:rating` ZSET fed by the Stage 3 rating consumer.
- [ ] Namespaces documented as rebuildable in `maichess-deploy`.
- [ ] REST contract handoff if a leaderboard endpoint is added.
- [ ] Tests to 100%; Stryker exclusions mirrored.
