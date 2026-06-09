# 09 — Caching Stage 1: immutable finished-match read model (Redis)

> Read `conventions.md` and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md) first.
> This is the lowest-risk, highest-leverage caching stage: **no new infrastructure** (Redis and
> Kafka already ship). Do this before `10`–`13`.

## Goal

Cache **immutable** data — finished match documents and `ListUserMatches` page results — in
Redis, invalidated/maintained by `MatchEnded` (and the match-finished projection), never by
manual cache-aside guesswork.

## Why it's safe

Once a match reaches an ended status its document never changes, so the cache needs **no TTL and
no invalidation logic** beyond "a new match for this user finished." Both the data and its only
invalidation signal already exist (Redis live read model + `match.events`/`MatchEnded`).

## What to read first

- [event-driven-architecture.md](../../knowledge/architecture/event-driven-architecture.md)
  (the Redis read model, `MatchEnded`, the projector).
- match-manager `Services/MatchService.cs` (`ListUserMatchesAsync`, the match-end paths) and
  `Data/MatchRepository.cs` (`FindForUserAsync`, `GetByIdAsync`).
- The existing Redis usage/config in `maichess-deploy/helm` and any current Redis client wiring.

## Implementation

1. **Finished-match doc cache.** On read of a match that is in an ended status (`GetMatchAsync`
   / the summary path), serve from Redis `match:{id}` if present; else load from match-db, cache
   with **no expiry** (LRU eviction only). Only cache **ended** matches — never cache `ongoing`
   (that is the live read model's job).
2. **`ListUserMatches` page cache.** Cache page results keyed by
   `matches:user:{userId}:{statusFilter}:{page}:{pageSize}`. Serve from cache on hit.
3. **Event-driven invalidation.** Where the match-finished fact is handled
   (`MatchEnded` consumer / the projector), for each participant **and** the `created_by`
   initiator: write/refresh `match:{id}` and **evict that user's `matches:user:*` page keys**
   (so the newly-finished game appears). No other path mutates these caches.
4. Keep the cache **behind the repository/service seam** so callers are unchanged; the cache is an
   adapter concern, not business logic. Match the service's existing style (no new abstraction
   layers per match-manager `CLAUDE.md`).

> Identity note: depends on the canonical-id fix from `08`. Cache keys must use the **same**
> canonical user-id representation as the DB filter, or cache and store will disagree.

## Deployment (required)

- No new component, but add/confirm the **Redis connection config** for match-manager in
  `maichess-deploy/helm` (service URL, any auth) and document the new key namespaces
  (`match:{id}`, `matches:user:*`) and that they are **rebuildable** (drop-safe; repopulate from
  match-db on miss).
- Note expected memory and eviction policy (`allkeys-lru` acceptable since every key is
  rebuildable).

## Tests (mandatory)

- Unit-test the caching behaviour against a mocked Redis + the existing `IMatchRepository` mock:
  hit/miss, ended-only caching, page-key eviction on match-finished for white/black/`created_by`.
- 100% line/branch/method on non-excluded code; mirror exclusions into Stryker config. Run
  `dotnet test -p:CollectCoverage=true` per match-manager `CLAUDE.md`.

## Verify

1. Load Profile → Past Matches twice; second load is served from cache (observe via logs/metrics
   or a Redis key inspection), identical results.
2. Finish a new match; the user's Past Matches reflects it immediately (page cache evicted).
3. Flushing Redis does not lose data — the list rebuilds from match-db on next read.

## Checklist

- [ ] Read the two ADRs + the match-manager code.
- [ ] Ended-only doc cache + page cache behind the repo/service seam.
- [ ] `MatchEnded`-driven write + page eviction for white/black/created_by.
- [ ] Canonical-id keys (consistent with `08`).
- [ ] Redis config + key-namespace doc in `maichess-deploy`.
- [ ] Tests to 100%; Stryker exclusions mirrored.
- [ ] Decision recorded against [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md).
