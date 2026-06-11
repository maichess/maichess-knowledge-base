# Serialization: Avro → Protobuf on Kafka

**Status:** Accepted — **COMPLETE** as of the [Kafka task program](../../tasks/planned/kafka/README.md) task `09`.
**Supersedes:** the Avro/Schema-Registry serialization section of
[event-driven-architecture.md](event-driven-architecture.md).

> **End state (task `09`, contracts v0.10.0):** the Confluent Schema Registry is **removed**.
> Every topic is **raw Protobuf bytes** — producers write `msg.ToByteArray()` (C#) /
> `toByteArray` (ScalaPB) / `OutboundEvent.encode` (ts-proto); consumers parse with
> `T.Parser.ParseFrom(value)` / `parseFrom` / `OutboundEvent.decode(value)`. No 5-byte Confluent
> framing, no registry lookup, no Avro read arm anywhere. The last Avro-on-the-wire topic
> (`user.events.v1`, formerly a CDC `GenericRecord`) is now Protobuf, and the `schema-registry`
> Helm component + every `SCHEMA_REGISTRY_URL` env are gone. Schemas live solely in
> `maichess-api-contracts/protos/` and ship in `Maichess.PlatformProtos` /
> `@maichess/platform-protos` / `io.github.maichess:platform-protos`. One IDL — Protobuf — across
> the Kafka events and the surviving query RPCs.

> **Progress (task `01`):** the Protobuf event schemas exist in
> `maichess-api-contracts/protos/events/v1/` — one file per topic
> (`socket_outbound`, `matchmaking_events`, `match_commands`, `match_events`,
> `analysis_commands`, `analysis_events`, `user_events`, `cheat_events`), all in package
> `maichess.events.v1`, mirroring the `events/v1/*.avsc` field-for-field. `buf lint`/`build` pass and
> codegen emits C#/Scala/TS. Contracts **v0.6.0** is published, every consumer is pinned at it
> (`*.csproj`, `build.sbt`, `package.json`), and the per-language serde helpers + round-trip tests
> are written: C# `ProtobufEventSerdes.cs` (match-maker, match-manager), Scala
> `ProtobufEventSerdes.scala` (move-validator, engine), Node `src/kafka/protobuf-serde.ts`
> (socket-service). Serde plumbing only — **no producer/consumer is switched**; that is task `02`.
> Local build/test of each service is the remaining handoff (a fresh agent shell can't restore the
> published package from GitHub Packages without a token); the verify command per service is in its
> `CONTRACT_NOTES.md`.
>
> **Task `02` is done:** the three live topics — `socket.outbound.v1`, `matchmaking.events.v1`,
> `match.commands.v1` — are migrated to Protobuf. Producers emit only proto; consumers dual-read
> Avro *or* Protobuf (discriminating on the Confluent schema id's registry type) so the cutover is
> reversible, and each fire-and-forget consume path now WARN-logs decode failures. The three `.avsc`
> files are retired (canonical + every embedded copy). The socket caveat (match-manager's socket
> push reverted to gRPC, failing `socket.outbound` silently) is resolved: `Socket__Transport: kafka`.
> The Avro **read** arms remain until the registry is removed in task `09`. No new Avro is on the
> wire; the remaining `.avsc` belong to topics that have not shipped a producer yet
> (`match.events`, `user.events`, `analysis.*`, `cheat.events`, `matchmaking.commands`).

## Context

The first Kafka topics shipped on **Avro + Confluent Schema Registry** (BACKWARD compat). The
project's historical and intended contract format is **Protobuf** (the surviving query RPCs and
all of `maichess-api-contracts/protos/` are already proto). We want one IDL — Protobuf — across
both the synchronous RPCs and the Kafka events.

## Decision

**Protobuf-first.** All event serialization is Protobuf; the Schema Registry is **removed at the
end** in favor of raw Protobuf bytes. Concretely:

- **New event types are authored in Protobuf from the start** — never written in Avro and
  re-encoded. (The move loop, analysis, rating, and cheat events are all born in proto.)
- **The handful of topics that shipped first on Avro** (`socket.outbound`, `matchmaking.events`,
  `match.commands`) are re-encoded to Protobuf per-topic, dual-read → switch → retire.
- The registry stays (mediating Protobuf) until every topic is proto-only, then it is removed.

### Strategy for the already-Avro topics: incremental dual-serde, per topic

In the existing rollout order
(socket.outbound → matchmaking → match loop → analysis → user/rating → cheat):

1. **Add** the Protobuf schema for the topic's events in `maichess-api-contracts` (under
   `protos/events/...`), mapping the common envelope fields
   (`event_id`, `event_type`, `aggregate_id`, `sequence`, `occurred_at`, `correlation_id`,
   `causation_id`, `producer`, `payload`) to a proto envelope message with a `oneof payload`.
2. **Consumers read both.** Update consumers to deserialize Avro *or* Protobuf (discriminate by
   the serde's magic byte / registry id during the transition).
3. **Producers switch** to Protobuf once all of a topic's consumers read both.
4. **Retire** the topic's Avro schema and the dual-read path.

This is safe and reversible at every step and never requires a synchronized multi-language
cutover. Transitional dual-path code is the accepted cost.

### Schema Registry: keep during transition, remove at the end

- **During migration:** keep the Confluent Schema Registry and switch to its **Protobuf serde**.
  The registry supports Protobuf and preserves BACKWARD compatibility checks and schema evolution
  safety while both formats coexist — this is what makes the dual-read discrimination clean.
- **Final phase (DONE in task `09`, after every topic was Protobuf-only):** the Schema Registry
  was **removed**. Producers/consumers now use **raw Protobuf bytes** with schemas managed solely in
  `maichess-api-contracts` and generated per language. This was a deliberate end-state, not an
  accident — the registry came out all the way once it was no longer carrying Avro or mediating a
  dual format.

The registry removal was its own final, prompted step so the dual-format safety net stayed in place
right up until the last topic was converted.

### Per-language serdes

- **C# (.NET services):** `Google.Protobuf` + Confluent Protobuf serde during transition; raw
  proto deserialize after registry removal.
- **Scala (engine, move-validator):** zio-kafka with a Protobuf serde (ScalaPB-generated types)
  replacing the avro4s/Avro serde.
- **Node (socket, auth-adjacent):** `protobufjs` / ts-proto generated types replacing the Avro
  serde.

Generated Protobuf packages publish from `maichess-api-contracts` on a `v*` tag exactly like the
existing `Maichess.PlatformProtos` flow; consumers bump the pinned version per the standard
handoff.

### Publish-first: a proto change is not usable until the package is published

**Editing a `.proto` does nothing for the services on its own.** The generated types only reach a
service through the published `Maichess.PlatformProtos` / `@maichess/platform-protos` /
`io.github.maichess:platform-protos` package. So whenever `maichess-api-contracts/` changes
(new event schemas, field changes, anything under `protos/`):

1. **Publish first.** The user commits, tags `vX.Y.Z`, and pushes the contracts repo so CI builds
   and publishes the package. A fresh agent's shell **cannot** restore a just-published version
   (GitHub Packages auth / propagation lag), so this is always a human handoff — see
   [tasks/conventions.md](../../tasks/conventions.md) §2.
2. **Then bump consumers.** Reconcile the pinned version in **every** consuming service
   (`*.csproj`, `build.sbt`, `package.json`) to the newly published version, not just the one you
   touched — versions have drifted between services before.
3. **Then write the code** that uses the new types (serde glue, producers, consumers). It cannot
   compile or be tested until steps 1–2 land, so until then the work is documented as a blocker in
   each affected service's `CONTRACT_NOTES.md`.

### Known serde touch-points (kept localised on purpose)

Each topic's Avro serde is confined to a single class per service, so the per-topic switch (add
proto schema → consumers read both → producer switches → retire Avro) is a small, contained change:

- `match.commands.v1`: producer = match-maker `Queue/KafkaMatchCreator.cs` (the only producer; behind
  the `IMatchCreator` seam, so service logic is untouched by the swap); consumer = match-manager
  `Events/MatchCommandConsumer.cs` (the class that must learn to read both formats first).
- `socket.outbound.v1` / `matchmaking.events.v1`: match-maker `Queue/KafkaMatchmakingNotifier.cs`
  (producer) and socket-service `src/kafka/consumer.ts` (consumer).

## Deployment

- No new infra was added; the end state **removed** the `schema-registry` Helm component and its
  config (task `09`). It is no longer deployed in any environment.
- Topic names/keys/partitions are unchanged — this is a payload-encoding migration, not a
  topology change. (If a parallel `vN` topic is ever needed for a non-back-compatible event, that
  is handled per-topic, but the default is in-place re-encoding.)

## Non-goals

- The synchronous query RPCs are already Protobuf; they are untouched.
- This does not change event semantics, ordering, idempotency, or the envelope's field set —
  only its wire encoding and the IDL it is generated from.
