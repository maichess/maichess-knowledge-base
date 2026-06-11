# 18 — Bot arena Kafka-native completion (replace `CollectionPoller`)

> Read `conventions.md`,
> [event-driven-architecture.md](../../knowledge/architecture/event-driven-architecture.md), and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md) first.
> Depends on Kafka task `06` (match events on `match.events.v1`). No new topics or contracts.

## Goal

Replace the 2-second polling loop (`CollectionPoller`) with a Kafka consumer that reacts to
`MatchEnded` events from `match.events.v1`. This eliminates polling latency entirely, removes
N serial/parallel gRPC calls per tick, and makes the bot arena a first-class event-driven
consumer.

## Background

`CollectionPoller` (`services/maichess-bot-arena-service/Arena/CollectionPoller.cs`) currently:
1. Calls `store.ListRunningGamesAsync` every 2 seconds.
2. Fans out `reader.ReadAsync(game.MatchId)` in parallel to match-manager (gRPC).
3. For any game with an outcome, calls `service.HandleFinishedGameAsync`.

After the Kafka match command side (task 06), `match.events.v1` carries `MatchEnded` events for
every match, including bot-vs-bot matches. The arena can react directly to those events instead
of polling.

## Implementation

### 1. Add a Kafka consumer to bot-arena-service

Mirror the consumer pattern used in match-manager's `MatchEventProjectorConsumer.cs`:

- Create `Kafka/ArenaMatchCompletionConsumer.cs` — an `IHostedService` that consumes
  `match.events.v1` (consumer group `bot-arena-completion`), deserialized as
  `Maichess.Events.V1.MatchEvent` (Protobuf, same as match-manager).
- On each `MatchEnded` message, look up the `matchId` in `store.TryGetRunningGameAsync(matchId)`.
  If the game exists in the arena's running-game list, map the `MatchEnded` payload to a
  `MatchOutcome` and call `service.HandleFinishedGameAsync(game, outcome, ct)`.
- If the game is not tracked by the arena, skip (the arena only cares about games it spawned).

### 2. Retire `CollectionPoller`

- Remove `CollectionPoller.cs` and its `AddHostedService<CollectionPoller>()` registration in
  `Program.cs`.
- Add `AddHostedService<ArenaMatchCompletionConsumer>()` in its place.
- Keep `IMatchOutcomeReader` if it is still used for other purposes; otherwise remove it too.

### 3. Add `TryGetRunningGameAsync` to `IArenaStore`

If `IArenaStore` does not already have a by-matchId lookup (it currently has
`ListRunningGamesAsync`), add `TryGetRunningGameAsync(string matchId, CancellationToken ct)`.
Implement in `ArenaStore` via a filtered `Query` to the database service. Apply
`[ExcludeFromCodeCoverage]` per the infrastructure exclusion rule.

### 4. Helm configuration

Add the Kafka consumer env vars to `botArenaService` in `maichess-deploy/helm/maichess/values.yaml`:
```yaml
Kafka__BootstrapServers: "{{ .Values.kafka.bootstrapServers }}"
Kafka__ConsumerGroup: "bot-arena-completion"
```
Mirror the pattern used in match-manager's Helm values.

## Tests (mandatory)

`CollectionPoller` is already `[ExcludeFromCodeCoverage]` (fire-and-forget background service),
and `ArenaMatchCompletionConsumer` will be the same. The testable seam is the decision logic:

- Unit-test a `ArenaMatchCompletionProjection` (pure function / static helper) that maps a
  `MatchEnded` proto event + a lookup result to either a `HandleFinishedGame` command or a no-op.
- Test: matched game → produces command; untracked match → produces no-op; different event type →
  produces no-op.
- Ensure the `store.TryGetRunningGameAsync` seam is mocked in existing `CollectionService` tests
  if it overlaps; extend coverage as needed.

Run `dotnet test -p:CollectCoverage=true`.

## Verify

1. Start the service with Kafka running; spawn a bot-vs-bot arena game; confirm the arena
   registers the result within seconds of `MatchEnded` being produced, without any polling logs.
2. No `ListRunningGamesAsync + ReadAsync` fan-out visible in tracing.
3. Coverage passes.

## Checklist

- [ ] `ArenaMatchCompletionConsumer` implemented as `IHostedService` (`[ExcludeFromCodeCoverage]`).
- [ ] Decision core unit-tested (projection helper — matched/untracked/wrong-type).
- [ ] `CollectionPoller.cs` removed; `Program.cs` registration updated.
- [ ] `IArenaStore.TryGetRunningGameAsync` added if absent; `ArenaStore` impl excluded from coverage.
- [ ] Helm values: Kafka consumer group added to `botArenaService`.
- [ ] `dotnet test -p:CollectCoverage=true` passes.
- [ ] Update `caching-and-read-models.md` "Known gaps" to remove the arena polling entry.
