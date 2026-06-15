# Kafka 01 — Protobuf event foundation

> Read this program's [README](README.md) and
> [serialization-protobuf-migration](../../../knowledge/architecture/serialization-protobuf-migration.md)
> first. **Depends on:** nothing in code, but ends with a **contracts publish handoff**.

## Goal

Stand up everything needed to produce and consume **Protobuf** events on Kafka — the contract
schemas and the per-language serde glue — **without switching any producer or consumer yet**. This
unblocks every later task: existing topics migrate to proto (`02`), and all new move-loop/analysis/
rating events are authored directly in proto.

## Deliverables

### 1. Proto event schemas in `maichess-api-contracts/protos/events/v1/`

One `.proto` per topic, each a common **envelope message** with a `oneof payload`, mirroring the
existing Avro `.avsc` in `maichess-api-contracts/events/v1/` **field-for-field** (same names, same
semantics — this is a re-encoding, not a redesign):

- `socket_outbound.proto`, `matchmaking_events.proto`, `match_commands.proto` — match the live Avro.
- `match_events.proto` — the move-loop events (`MatchCreated`, `MoveSubmitted`, `MoveValidated`,
  `MoveRejected`, `MoveApplied`, `BotMoveRequested`, `BotMoveCalculated`, `DrawOffered`,
  `DrawDeclined`, `MatchEnded`) per the `match.events.v1.avsc` already embedded.
- `analysis_commands.proto`, `analysis_events.proto`, `user_events.proto`, `cheat_events.proto` —
  mirror their `.avsc`.

Envelope fields on every message: `event_id`, `event_type`, `aggregate_id`, `sequence`,
`occurred_at`, `correlation_id`, `causation_id`, `producer`, then `oneof payload { … }`. Follow the
proto style in `maichess-api-contracts/CLAUDE.md` (snake_case fields, PascalCase messages,
SCREAMING_SNAKE enums, `v1/` versioning). Avro enums → proto enums; Avro `union[null,T]` for optional
identity fields → proto `optional`/wrapper as appropriate.

> Keep the `.avsc` files in place for now — `02` retires each one as its topic cuts over.

### 2. Generated packages

Wire `buf generate` (or the existing codegen) so the new protos emit into the C#/Scala/TS targets and
ship in `Maichess.PlatformProtos`. **Publish handoff:** stop and have the user tag/push `vX.Y.Z`,
then bump the pinned version in **every** consuming service (`*.csproj`, `build.sbt`, the client/socket
`package.json`). Document the blocker in each touched service's `CONTRACT_NOTES.md` if you can't
restore the new version locally.

### 3. Per-language serde helpers (no behavior change)

Add the Protobuf serde plumbing behind the existing seams, but **do not switch any producer/consumer**:

- **C#** (match-manager, match-maker, …): a `Confluent.SchemaRegistry.Serdes.Protobuf`
  serializer/deserializer helper alongside the current Avro one. Reference `Google.Protobuf`.
- **Scala** (move-validator, engine): zio-kafka serde from ScalaPB-generated types (replaces the
  avro4s path planned in those services' `CONTRACT_NOTES`).
- **Node** (socket-service): ts-proto types + the Confluent Protobuf deserializer next to the Avro one.

## Tests

- Envelope + every `payload` variant round-trips through the generated proto types (encode→decode)
  in each language's test suite.
- Serde helper unit tests (serialize→deserialize a representative message per topic).
- Coverage/exclusions per the program README; `[ExcludeFromCodeCoverage]` only the I/O serde wrappers.

## Verify

- `buf lint` + `buf breaking` clean; packages generate for all three languages.
- A throwaway round-trip test per language deserializes a proto-encoded envelope to the right payload.
- No topic has switched yet — staging traffic is unchanged (still Avro on the wire).

## Docs to update

- `serialization-protobuf-migration` — note the proto schemas now exist and the serdes are wired.
- Each service `CONTRACT_NOTES.md` that gains the proto serde / version bump.
