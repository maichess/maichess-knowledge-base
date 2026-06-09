# Serialization: Avro → Protobuf on Kafka

**Status:** Accepted — implemented as part of the [Kafka task program](../../tasks/planned/kafka/README.md)
**Supersedes (eventually):** the Avro/Schema-Registry serialization section of
[event-driven-architecture.md](event-driven-architecture.md).

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
- **Final phase (explicit, after every topic is Protobuf-only):** **remove the Schema Registry**.
  Producers/consumers switch to **raw Protobuf bytes** with schemas managed solely in
  `maichess-api-contracts` and generated per language. This is a deliberate end-state, not an
  accident — the registry comes out all the way once it is no longer carrying Avro or mediating a
  dual format.

The registry removal is its own final, prompted step so the dual-format safety net is in place
right up until the last topic is converted.

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

### Known serde touch-points (kept localised on purpose)

Each topic's Avro serde is confined to a single class per service, so the per-topic switch (add
proto schema → consumers read both → producer switches → retire Avro) is a small, contained change:

- `match.commands.v1`: producer = match-maker `Queue/KafkaMatchCreator.cs` (the only producer; behind
  the `IMatchCreator` seam, so service logic is untouched by the swap); consumer = match-manager
  `Events/MatchCommandConsumer.cs` (the class that must learn to read both formats first).
- `socket.outbound.v1` / `matchmaking.events.v1`: match-maker `Queue/KafkaMatchmakingNotifier.cs`
  (producer) and socket-service `src/kafka/consumer.ts` (consumer).

## Deployment

- No new infra to *add*; the end state **removes** the `schema-registry` Helm component and its
  config once migration completes. Until then it stays deployed.
- Topic names/keys/partitions are unchanged — this is a payload-encoding migration, not a
  topology change. (If a parallel `vN` topic is ever needed for a non-back-compatible event, that
  is handled per-topic, but the default is in-place re-encoding.)

## Non-goals

- The synchronous query RPCs are already Protobuf; they are untouched.
- This does not change event semantics, ordering, idempotency, or the envelope's field set —
  only its wire encoding and the IDL it is generated from.
