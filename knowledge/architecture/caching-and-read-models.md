# Caching & Read Models (CQRS)

**Status:** Accepted — implementation staged (see `tasks/09`–`13`)
**Builds on:** [event-driven-architecture.md](event-driven-architecture.md) (Kafka, the
Redis live-match read model, the 202 + socket flow). This ADR generalises the read/write
split that the match domain already uses to the rest of the platform.

## Context

The match domain is already CQRS: `match.events.v1` is the write side; a Redis projection +
the match-db materialisation are the read side. Everything else is still read-through-the-
master-store: `GetUser` is a synchronous RPC on a hot path, finished matches and the games
library are re-fetched from Mongo on every view, and there is no leaderboard structure.

Two facts shape the strategy:

1. **Finished matches and analysis games are immutable.** Once a match ends or a game is
   imported, the document never changes. Immutable data caches with infinite TTL and *no*
   invalidation logic.
2. **Kafka already carries every fact that would invalidate a cache.** `MatchEnded`, the rating
   event, and the user/cheat events are the exact signals a cache needs. So caches are
   maintained by **consuming events**, not by cache-aside guesswork.

## Decision

Adopt a uniform pattern: **every read model is a projection fed by events; Redis is the default
cache tier; invalidation is event-driven, never manual.** No service invalidates another
service's cache by reaching across a boundary — it consumes the fact and updates its own model.

### Cache map

| Read model | Tier | Mutability | Maintained by |
|---|---|---|---|
| Live match state | Redis | mutable | `match.events` projector (existing) |
| Finished match doc + `ListUserMatches` pages | Redis | **immutable** | `MatchEnded` consumer; infinite TTL, evict on LRU only |
| User profile/rating/stats replica | Redis (default) | mutable | compacted `user.events.v1` consumer |
| User rating enrichment in **Match Maker** | Streamiz KTable | mutable | co-partitioned `user.events` ⋈ matchmaking join |
| `analysis_results` hot positions | Redis L1 over Mongo L2 | append-only | cache-aside, keyed `(fen, bot_id)` |
| Leaderboards / rating ranges | Redis **sorted set** (`ZSET`) | mutable | rating event consumer |
| Arena standings | Redis | recomputed | tournament event consumer |
| Games / match / position **search** | Elasticsearch | derived | see [search-elasticsearch.md](../services/search-service.md) |

### Redis vs KTable — the deliberate split

Both materialisation styles are in the codebase **on purpose**, so they can be compared in
practice. The rule for choosing:

- **Redis-materialised replica is the default.** A plain consumer reads the compacted
  `user.events.v1` from the beginning into shared Redis (`user:{id}`); services read Redis
  instead of calling `GetUser`. Simple, one mental model, reuses infra we already deploy.
- **Streamiz KTable is used in exactly one place: Match Maker.** Matchmaking commands are keyed
  by `playerId` and `user.events` is keyed by `userId`, so they **co-partition** — the canonical
  Kafka Streams stream-table join. The KTable state is bounded per partition (no GlobalKTable /
  full replication), and the enrichment need is real (skill-based pairing reads live ratings).
  This is the spot where the KTable's downsides (heavy dep, per-pod state) are smallest and its
  payoff is largest. Do **not** spread Streamiz to services where `user.events` does not
  co-partition with the input stream — there it would force a GlobalKTable (full replica on every
  pod), which is strictly worse than the Redis replica.

### Invalidation is a consumed fact

Examples of the one rule:
- `MatchEnded` → the ending service's projector writes the finished-match cache and busts the
  participant/initiator's `ListUserMatches` pages.
- rating event on `user.events` → refresh `user:{id}`, update the leaderboard `ZSET`.
- `cheat.events` → update the user replica's `flagged` field (see
  [anticheat-service.md](../services/anticheat-service.md)).

No TTL-based "eventually correct" caches for mutable data; correctness comes from the event, not
from expiry.

### Stage 1 implementation notes (finished-match read model — `tasks/09`)

Implemented in match-manager (`tasks/09`). Concrete choices made:

- **Service seam, not a new layer.** Caching lives behind an `IMatchCache` adapter
  (`Data/IMatchCache.cs`, Redis impl `Data/RedisMatchCache.cs`) injected into `MatchService`,
  mirroring the existing `IMatchRepository`/`MatchRepository` split. `RedisMatchCache` is
  `[ExcludeFromCodeCoverage]` (needs live Redis); the orchestration in `MatchService` is unit-tested
  against a mocked `IMatchCache`. REST/gRPC callers are unchanged.
- **Invalidation is inline, not a separate consumer.** match-manager *is* the service that ends a
  match, so the "MatchEnded fact" is handled at the existing end paths (move-ends-game, resign,
  accept-draw, timeout watchdog, external sync) via one private `OnMatchEndedAsync`, rather than a
  second Kafka consumer re-reading `match.events` the same pod just produced. It refreshes
  `match:{id}` and evicts the page cache for white, black, and `created_by` (deduplicated, canonical
  ids).
- **Only the *ended* page is cached.** `ListUserMatches` caches the `ended` status filter only
  (`matches:user:{userId}:ended:{page}:{pageSize}`); the `ongoing` filter always reads live, since
  an ongoing list changes on match *creation* (not a signal this cache consumes). The finished-match
  doc cache (`match:{id}`) is likewise written only for non-`ongoing` statuses.
- **Per-user SCAN for eviction**, not a tracked index set: under `allkeys-lru` an index could be
  evicted independently of the page keys it references, leaking stale entries. The `matches:user:{id}:*`
  pattern keeps the scanned keyspace small.
- **Canonical ids** (from `tasks/08`) are used for every cache key and eviction, so the
  cache and the DB filter agree.

### Stage 3 implementation notes (user read models — `tasks/11`)

Both materialisations of the compacted `user.events.v1` are now in the codebase, on purpose:

- **Redis replica (default), in match-manager.** A plain Kafka consumer
  (`Kafka/UserReplicaConsumer.cs`, `[ExcludeFromCodeCoverage]`) reads `user.events.v1`
  from the beginning through a pure projection (`Kafka/UserReplicaProjection.cs`) into the
  shared Redis hash `user:{id}`, behind the `Data/IUserReplica` seam (Redis impl
  `Data/RedisUserReplica.cs`). The two hot `GetUser` callers — match-end rating enrichment
  (`MatchService.ResolveOpponentRatingAsync`) and username resolution
  (`MatchService.ResolveUsernameAsync`, used by the REST player-response mapping) — read the
  replica first and fall back to `GetUser` on a cold miss. Replica fields are **nullable**,
  so a partially-warmed row (e.g. only a `UserRegistered` snapshot, no rating yet) still
  defers to `GetUser` rather than rating against a default. Rebuildable by resetting the
  `match-manager-user-replica` consumer group. The same `user:{id}` namespace will carry
  leaderboard (Stage 4) and `flagged` (anti-cheat) fields.

- **Streamiz KTable (the one KTable), in Match Maker only.** `Streaming/UserRatingTopology.cs`
  folds `user.events.v1` (keyed by userId) into a RocksDb-backed KTable of live ratings
  (`user-ratings-store` + its changelog) via an aggregate — an aggregate, not `MapValues`,
  because only `RatingUpdated` carries a rating and a profile-only update must not clobber
  it. `matchmaking.events.v1` (`PlayerEnqueued`, keyed by playerId) **co-partitions** with
  it (both 3 partitions), so an inner KStream-KTable **join** tags each enqueue with the
  live rating; the inner join naturally excludes users the KTable does not yet know.
  Skill-based pairing (`MatchingService`) reads the store locally via interactive queries
  (`IUserRatingStore`), pairs the minimum rating gap, and falls back to FIFO when the pool
  is small or a rated pair cannot be claimed. The topology is unit-tested with Streamiz's
  `TopologyTestDriver`. Streamiz stays in Match Maker only: anywhere `user.events` does not
  co-partition with the input stream it would force a GlobalKTable (full per-pod replica) —
  strictly worse than the Redis replica.

- **Contract.** `user.events.v1`'s `RatingUpdated` payload gained `wins/losses/draws`
  (defaults, backward-compatible) so the replica can carry stats; the User-service CDC
  mapper (`tasks/10`) populates them. No `PlatformProtos` bump (Avro schema, not
  gRPC).

## Deployment

- Redis already ships in Helm; new caches reuse it (note future sharding if `ZSET`/replica load
  grows — out of scope here).
- The Match Maker KTable needs a Streamiz state directory (RocksDB) on the pod; size and
  changelog topic are documented in `tasks/11`.
- No cache is a system of record. Every read model is rebuildable from its topic (replay) or its
  master store; document the rebuild path with each projection.

## Non-goals

- This ADR does not introduce CDC; that boundary lives in
  [change-data-capture.md](change-data-capture.md).
- Elasticsearch scope lives in [search-elasticsearch.md](../services/search-service.md).
