# 17 — `ListBots` in-memory cache in match-manager

> Read `conventions.md` and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md) first.
> No new infrastructure required. Independent of other open tasks.

## Goal

Eliminate the per-match-creation gRPC roundtrip to the engine service for `ListBots`.
`MatchService` currently calls `engineClient.ListBotsAsync` on every bot-vs-bot match creation to
validate the requested bot IDs. The bot list is entirely static between deploys — there is no
mechanism to add bots at runtime. Caching it with a long-TTL `IMemoryCache` entry removes the
hot-path gRPC call with no correctness risk.

## Background to read first

- `services/maichess-match-manager-service/Match/MatchService.cs` — the `CreateBotVsBotMatchAsync`
  path (and any other caller of `ListBotsAsync`).
- `services/maichess-match-maker-service/Queue/QueueingService.cs` — also calls the engine client
  for bot lookups; apply the same fix there if confirmed.
- The engine service's `ListBots` gRPC endpoint
  (`maichess-api-contracts/proto/engine-service/v1/bots.proto`).

## Implementation

1. Inject `IMemoryCache` into `MatchService` (it is already registered in match-manager's DI
   because `ArenaSettingsService` uses it — check `Program.cs` to confirm, and add
   `builder.Services.AddMemoryCache()` if absent).
2. Introduce a private helper method `GetBotListAsync(CancellationToken)` that:
   - Returns the cached `IReadOnlyList<BotInfo>` on a hit.
   - Falls through to `engineClient.ListBotsAsync` on a miss, caches the result with a 10-minute
     TTL, and returns it.
   - Cache key: `"engine:bots"`.
3. Replace every `engineClient.ListBotsAsync` call in `MatchService` with `GetBotListAsync`.
4. If `QueueingService` in match-maker has the same pattern, apply identically (it has its own
   `IMemoryCache` instance; same cache key is fine since they are separate processes).

## Tests (mandatory)

The bot-list fetch is already indirectly covered by the existing `MatchService` tests. Add
specific cases:

- Cache hit path: second `CreateBotVsBotMatchAsync` call does **not** invoke `engineClient.ListBotsAsync` again.
- Cache miss path: first call invokes `engineClient.ListBotsAsync` exactly once.
- Expired cache: after TTL-equivalent time, next call fetches fresh.
- Use `MemoryCache` directly (real, not mocked) as in `SettingsContext` in the arena service tests.

Run `dotnet test -p:CollectCoverage=true` to verify 100% on non-excluded code.

## Verify

1. Start the service locally; trigger two rapid bot-vs-bot match creations; confirm in logs that
   `ListBots` gRPC is called once, not twice.
2. Coverage passes.

## Checklist

- [ ] `IMemoryCache` injected into `MatchService` (and `QueueingService` if applicable).
- [ ] `GetBotListAsync` helper with 10-minute TTL, cache key `"engine:bots"`.
- [ ] All `engineClient.ListBotsAsync` call sites replaced.
- [ ] Tests: hit/miss/expiry scenarios.
- [ ] `dotnet test -p:CollectCoverage=true` passes.
- [ ] Update `caching-and-read-models.md` "Known gaps" to remove `ListBots` entry.
