# 11 — Caching Stage 3: user read models (Redis replica + Match Maker KTable)

> Read `conventions.md` and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md) first.
> **Depends on `10`** (the `user.events` topic must be CDC-derived/trustworthy first).

## Goal

Kill the hot synchronous `GetUser` RPC by materialising the compacted **`user.events.v1`** into
local read models. Two materialisations on purpose, so they can be compared:

1. **Redis-materialised replica (the default, used everywhere).**
2. **Streamiz KTable (used in exactly one place: Match Maker).**

## Part A — Redis replica (default)

1. A plain Kafka consumer reads compacted `user.events.v1` **from the beginning** into shared
   Redis `user:{id}` → `{ username, rating, rating_deviation, wins, losses, draws, flagged, … }`.
2. Services that currently call `GetUser` on a hot path (e.g. Match Manager's match-end rating
   enrichment / username resolution in `ToPlayerResponseAsync`) read **Redis** instead. Keep a
   `GetUser` fallback for cold misses while the replica warms.
3. Updates are event-driven: every `user.events` record upserts `user:{id}`. The replica is
   **rebuildable** by replaying the compacted topic.

> This also lets later stages (leaderboard `ZSET`, anti-cheat `flagged`) ride the same replica.

## Part B — Match Maker KTable (the one KTable)

Match Maker is the single place a KTable's downsides are smallest and its payoff largest:
matchmaking commands are keyed by `playerId` and `user.events` by `userId`, so they
**co-partition** — a bounded, per-partition stream-table join (no GlobalKTable / full
replication).

1. Add **Streamiz** (.NET Kafka Streams) to Match Maker.
2. Build a **co-partitioned KTable** of `user.events` (rating + `flagged`) and **join** it onto
   the matchmaking stream so skill-based pairing reads **live ratings locally**, no `GetUser` RPC.
3. State store: RocksDB-backed, with its changelog topic. Document the state dir + changelog.
4. **Do not** spread Streamiz to other services: where `user.events` does not co-partition with
   the input stream it would force a GlobalKTable (full per-pod replica) — strictly worse than the
   Redis replica. Those services use Part A.

## Deployment (required)

- **Redis replica consumer:** deploy as part of (or alongside) the consuming services; document
  the `user:{id}` namespace and that it is rebuildable from `user.events`.
- **Match Maker KTable:** add a persistent volume / state directory for RocksDB on the Match
  Maker pod; provision the **KTable changelog topic** in the topic-init job; document partition
  co-location requirements (matchmaking topic and `user.events` must share partition count for the
  co-partitioned join).
- Note the Streamiz dependency in Match Maker's project + `README`.

## Contracts

- No new event contract (consumes existing `user.events.v1`). If a field needed by the replica is
  missing from the `user.events` payload, add it via the standard api-contracts publish/bump
  handoff.

## Tests (mandatory)

- Redis replica consumer: upsert on `user.events`, cold-miss `GetUser` fallback, rebuild-from-
  replay. 100% on non-excluded code.
- Match Maker: test the stream-table join enriches a pairing with the KTable's rating and excludes
  appropriately; use Streamiz's `TopologyTestDriver`-equivalent test harness. Mirror Stryker
  exclusions.
- Run `dotnet test -p:CollectCoverage=true` for each touched service.

## Verify

1. With the replica warm, the hot `GetUser` call disappears from the match-end / matchmaking paths
   (observe via traces — the Kafka hub renders instead of the RPC, per the observability note in
   the event-driven ADR).
2. A rating change in `user.events` is reflected in both the Redis replica and the Match Maker
   KTable without any service calling `GetUser`.
3. Restart Match Maker: the KTable restores from its changelog; restart the replica consumer: it
   rebuilds from the compacted topic.

## Checklist

- [ ] Redis replica consumer materialising `user:{id}` from compacted `user.events`.
- [ ] Hot `GetUser` callers switched to the replica (with cold-miss fallback).
- [ ] Streamiz co-partitioned KTable join in **Match Maker only**.
- [ ] State dir + changelog topic + co-partition requirements deployed/documented.
- [ ] Tests to 100%; Stryker exclusions mirrored.
- [ ] Rationale recorded against [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md).
