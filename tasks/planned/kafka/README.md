# Kafka migration — Protobuf-first program

An ordered set of self-contained tasks that take the match domain to its target state: an
**event-sourced Kafka platform serialized with Protobuf**, a Redis live read model, a 202 move
contract, and no dead gRPC. Read the design first:
[event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md) (target
topology, topics, envelope, match-flow) and
[serialization-protobuf-migration](../../../knowledge/architecture/serialization-protobuf-migration.md)
(why Protobuf, registry-then-no-registry).

**Run them in order** (dependencies below). Each `NN-*.md` is one working session. The shared rules
in [../../conventions.md](../../conventions.md) and the conventions in this file apply to all of them.

## Guiding decision: Protobuf-first, no new Avro

The end state is Protobuf on every topic with the Schema Registry eventually removed. Therefore:

- **New event types are authored in Protobuf from the start** — never written in Avro and re-encoded.
  All the move-loop events (`MoveSubmitted`, `MoveValidated`, `MoveApplied`, `BotMoveRequested`,
  `BotMoveCalculated`, `MatchEnded`, …) are born in proto.
- **The three topics already live on Avro** migrate per-topic, dual-read → switch producer → retire
  Avro (task `02`). These are the only Avro that ever existed on the wire.
- The Confluent **Schema Registry stays** (switched to its Protobuf serde) until the final task, which
  removes it in favor of raw Protobuf bytes.

## Current state (what already runs on Kafka)

Live today, **in Protobuf** (these three migrated off Avro in task `02`), gated by the
`Socket:Transport` config flag (`kafka` is the code default; Kafka is `enabled` in staging only —
prod has `kafka.enabled: false`):

- **`socket.outbound.v1`** client fan-out — socket-service consumes (`src/kafka/consumer.ts`, dual-read)
  and fans out over socket.io. Producers: match-manager `Events/KafkaSocketNotifier.cs`, match-maker
  `Queue/KafkaMatchmakingNotifier.cs`.
- **`matchmaking.events.v1`** — match-maker publishes `PlayersMatched` (`Queue/KafkaMatchmakingNotifier.cs`);
  the ratings topology (`Streaming/UserRatingTopology.cs`) consumes it (dual-read). Matchmaking status
  is still **polled** by the client, so the `matched` push is not yet load-bearing.
- **`match.commands.v1`** `CreateMatchCommand` — match-maker `Queue/KafkaMatchCreator.cs` produces;
  match-manager `Events/MatchCommandConsumer.cs` materialises the match with the caller-minted id
  (dual-read). Bot-vs-bot creation stays synchronous gRPC (needs `start_fen` validation).

The consumers **dual-read** Avro and Protobuf (discriminating on the Confluent schema id's registry
type) so already-enqueued Avro messages still decode and the cutover is reversible; nothing *produces*
Avro any more, and the three `.avsc` files are retired. The Avro read arm is removed in task `09`
(registry removal). The socket caveat — match-manager's socket push had been reverted to gRPC,
failing the `socket.outbound` hop silently — is **resolved**: `Socket__Transport: kafka` and every
fire-and-forget consume path now WARN-logs a decode failure instead of dropping silently.

Everything else is **synchronous gRPC**: the whole move loop
(`MatchService.MakeMoveAsync`/`ProcessBotMoveAsync` → `MovesClient.ValidateMove`,
`BotsClient.GetBestMove`, `UserServiceClient.RecordMatchResult`, writes to match-db, then broadcast);
analysis (logic still in **maichess-mono**); ratings. The Redis **finished-match** cache
(`RedisMatchCache`) and user replica exist, but there is **no live read model** for ongoing matches.
`match.events.v1` has **no producer** yet.

Infra exists: Helm `kafka`, `schema-registry`, `kafka-topics-init` (topics already created).

## Tasks

| # | Task | Touches | Depends on |
|---|------|---------|-----------|
| 01 | [Protobuf event foundation](01-protobuf-event-foundation.md) — proto event schemas for every topic + per-language serdes | api-contracts, all services (serde glue) | — (needs a contracts publish) |
| 02 | [Migrate live topics to Protobuf](02-migrate-live-topics-to-protobuf.md) — socket.outbound, matchmaking, match.commands; resolve the socket caveat | socket-service, match-manager, match-maker | 01 |
| 03 | [Move-validator stream processor](03-move-validator-stream-processor.md) — `MoveSubmitted` → `MoveValidated`/`MoveRejected` | move-validator (Scala) | 01 |
| 04 | [Engine stream processor](04-engine-stream-processor.md) — `BotMoveRequested` → `BotMoveCalculated` | engine (Scala) | 01 |
| 05 | [Match read model + projector](05-match-read-model-and-projector.md) — Redis live read model; project `match.events` → state + `MoveApplied`/`MatchEnded`/`BotMoveRequested` | match-manager | 01, 03, 04 |
| 06 | [Match command side + 202](06-match-command-side-and-202.md) — `POST /moves`/`/resign` → `MoveSubmitted`, 202; draws; timeout watchdog → `MatchEnded{TIMEOUT}`; close the loop | match-manager, contracts, client | 05 |
| 07 | [Analysis over Kafka](07-analysis-over-kafka.md) — `analysis.commands`/`analysis.events`; engine streams depth updates | engine, maichess-mono/analysis | 01, 04 |
| 08 | [Rating events; retire RecordMatchResult](08-rating-events-and-retire-recordmatchresult.md) — emit rating updates on match end via `user.events` | match-manager, user-service | 01, 06 |
| 09 | [Decommission gRPC + remove the registry](09-decommission-grpc-and-remove-registry.md) — drop dead RPCs; raw Protobuf bytes; remove schema-registry | all services, maichess-deploy | 02, 06, 07, 08 |

After `06` the live match flow is fully event-sourced on Protobuf; after `09` the platform is at the
target state.

## Conventions for every task here

- **Envelope** on every message: `event_id`, `event_type`, `aggregate_id` (partition key), `sequence`
  (per-aggregate monotonic), `occurred_at`, `correlation_id`, `causation_id`, `producer`, `payload`
  (`oneof`). Mirror the field set exactly; map the Avro `union` to a proto `oneof payload`.
- **Serialization:** Protobuf via the Confluent **Protobuf** serde + Schema Registry (`BACKWARD`
  compat) **until task `09`**, which switches to raw Protobuf bytes and removes the registry. Proto
  schemas live in `maichess-api-contracts/protos/events/v1/`; generated types ship in
  `Maichess.PlatformProtos` (C#/Scala/TS) on the standard `v*` tag handoff.
- **Per-language serdes:** C# `Google.Protobuf` + `Confluent.SchemaRegistry.Serdes.Protobuf`; Scala
  zio-kafka + ScalaPB; Node ts-proto + the Confluent proto deserializer. Keep the existing transport
  seams (`IMatchmakingNotifier`, `IMatchCreator`, the socket consumer) so the serde swap is localized.
- **Idempotency/ordering:** keyed by `aggregate_id` for per-aggregate order; dedupe on
  `(aggregate_id, sequence)`; the validator is pure (safe to reprocess); the engine dedupes on
  `request_id`; use Kafka transactions on the validator and projector consume→produce steps for
  effectively-once.
- **Contract-publish handoff:** any change under `maichess-api-contracts/` stops to have the **user**
  tag/push a `vX.Y.Z`, then bumps `Maichess.PlatformProtos` in **all** consumers (see
  [../../conventions.md](../../conventions.md)). A fresh agent's shell cannot restore a freshly
  published version — implement, document the blocker in `CONTRACT_NOTES.md`, and hand off.
- **Observability:** keep Kafka client instrumentation on so `messaging.*` produce/consume spans flow
  (the topology graph renders the bus from OTel servicegraph spans).
- **Testing:** 100% line/branch/method on non-excluded code; mirror Stryker/Stryker4s exclusions;
  `[ExcludeFromCodeCoverage]` only the I/O glue (producers/consumers/serde wrappers), as today.
- **`DatabaseService.Insert`** honors a caller-supplied non-empty `id` (Mongo `_id`, duplicate →
  `AlreadyExists`; Postgres self-assigns).
