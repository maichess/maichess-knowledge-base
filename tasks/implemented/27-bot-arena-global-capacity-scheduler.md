# 27 — Bot arena: global capacity scheduler (start pending collections + react to concurrency changes)

> Read `conventions.md` and
> [bot-arena-service.md](../../knowledge/services/bot-arena-service.md) first.
> `maichess-bot-arena-service` only (no contract change needed for the core fix; an
> optional `GameResult` field is noted for the per-game tag). Fixes reports **17**
> (pending collection never starts), **13** (react to concurrency-limit change), and the
> backend half of **16** (live-vs-pending per-game tag).

## Root cause (confirmed in code)

`CollectionService.LaunchCollectionAsync(collection)` launches games for **one
collection only**. It is invoked from exactly two places:
`CreateAsync` (once, at creation) and `HandleFinishedGameAsync` (for the collection that
owns the finishing game). `CollectionPoller` only ever lists **running** games.

Consequences:
- **Item 17:** a collection created while the global cap is full is launched once with
  `LaunchPlanner.LaunchableCount(...) == 0`, stays `pending` with zero running games, and
  is **never retried** — the poller can't see it (no running games) and no finished game
  belongs to it. It sits pending forever and never starts or ends.
- **Item 13:** `ArenaSettingsService.SetConcurrencyLimitAsync` only persists + caches the
  new limit. Raising it launches nothing until some unrelated game happens to finish (and
  even then only refills the finishing game's own collection); lowering it does nothing
  active (acceptable — running games should finish naturally).

## Fix — one global reconcile step

Add a single serialized method, e.g. `CollectionService.FillAvailableCapacityAsync(ct)`,
that reconciles the **whole arena** against the global cap:

1. `cap = settings.GetConcurrencyLimit`; `running = store.CountRunningGamesAsync`.
2. While `running < cap`: pick the **oldest** collection with pending games (FIFO across
   collections — define the ordering explicitly, e.g. by collection `created_at`, then
   game `order`), launch its next pending game, promote it `pending → running`, increment
   `running`. Stop when capacity is full or no pending games remain.
3. Reuse `LaunchPlanner` for the per-step launchable math; keep launching one game at a
   time so the global `running` count stays accurate.

Invoke `FillAvailableCapacityAsync` from:
- `HandleFinishedGameAsync` (after advancing the finishing collection — replaces the
  single-collection relaunch, or runs in addition to it).
- `CreateAsync` (so a new collection starts immediately if capacity exists, else waits).
- `SetConcurrencyLimitAsync` (**on increase** — immediately fill the new headroom). On
  decrease, do nothing; in-flight games drain naturally until `running ≤ cap`.

**Serialize it.** Multiple games can finish in one 2-second poll tick; concurrent
reconciles could launch past the cap. Guard with the same per-tick sequential discipline
the poller already uses for `HandleFinishedGameAsync`, or a dedicated lock/semaphore.

Also: `ArenaSettingsService` caches the limit for 30s. Bust/refresh the cache inside
`SetConcurrencyLimitAsync` (it already `cache.Set`s) and make the reconcile read the fresh
value, so an increase takes effect at once rather than up to 30s later.

## Item 16 — per-game live vs pending tag (backend half)

The client now shows collection-level live/pending from `progress.running_games`
(done). For the **per-game** row tag, `GameResult` exposes only `result`
(`MATCH_STATUS_ONGOING` until end), so the UI can't tell "queued/pending" from
"actually playing". Optional contract addition: expose the arena game's own
`pending|running|finished` status (or a `started` bool) on `GameResult` so a queued game
shows "pending" and only a launched one shows "live". If added, follow the contract
publish/bump handoff and update the client `ArenaGameRow`.

## Testing

- Unit/BDD tests (this service uses Reqnroll feature files — see
  `MaichessBotArenaService.Tests/Features/ConcurrencyLimit.feature` and
  `CollectionLifecycle.feature`): 
  - A collection created at full capacity starts as soon as a running game finishes.
  - Two queued collections start FIFO as capacity frees.
  - Raising the limit immediately launches queued games up to the new cap.
  - Lowering the limit launches nothing; running count drains to ≤ cap.
  - Reconcile never exceeds the cap when several games finish in one tick.
- 100% on new non-excluded code; mirror exclusions into Stryker config. Watch the
  Reqnroll regex gotcha (`reqnroll-regex-vs-cucumber-gotcha`): anchor `(true|false)`-only
  step patterns with `^...$`.

## Implemented (2026-06-15)

- **Core fix (items 17 + 13).** `CollectionService.FillAvailableCapacityAsync(ct)`
  is the single global reconcile: `budget = LaunchPlanner.LaunchableCount(cap,
  running, pending)`, then it launches that many of the globally-pending games in
  FIFO order (oldest collection `created_at`, then game `order`), promoting each
  launched collection `pending → running`. Serialized with a `SemaphoreSlim`
  (`CollectionService` now `IDisposable`). It replaces the old per-collection
  `LaunchCollectionAsync` and is invoked from `CreateAsync`,
  `HandleFinishedGameAsync` (after advancing the finishing collection), and the new
  `CollectionService.SetConcurrencyLimitAsync` **on increase only**. New store
  primitive `IArenaStore.ListPendingGamesAsync` (filtered `status="pending"` query;
  excluded impl + fake). The `PUT /concurrency-limit` endpoint now routes through
  `CollectionService` (was `ArenaSettingsService`) so a raise fills headroom at
  once; `ArenaSettingsService` already cached the fresh value on set.
- **Item 16 (per-game tag).** REST `GameResult` gained a `status` field
  (`pending|running|finished`), set from `ArenaGame.Status` in `ResultViewBuilder`.
  Contract: `arena.proto` gained `ArenaGameStatus` enum + `GameResult.status = 8`
  (additive) and `rest/bot-arena.md` documents it — **published as contracts
  v0.13.0**. Arena is REST-only and does **not** consume `arena.proto`, so no .NET
  service needed a package bump for this. Client: `lib/models/arena.ts`
  (`ArenaGameStatus` + `status`) and `ArenaCollectionDetail` `GameLink` now tags
  "pending" vs "live" from `status` (and renders un-launched pending rows as static,
  non-watchable).
- **Tests:** new `Features/CapacityScheduler.feature` (7 scenarios) + steps; 104
  tests, **100% line/branch/method**. Verified against the existing contracts
  0.12.0 cache (no new package needed for the .NET build).
