# 15 — Migrate Kafka serialization: Avro → Protobuf

> Read `conventions.md`,
> [serialization-protobuf-migration.md](../../knowledge/architecture/serialization-protobuf-migration.md),
> and [event-driven-architecture.md](../../knowledge/architecture/event-driven-architecture.md)
> first. This is a **multi-session, per-topic** migration spanning C#, Scala, and Node — not a
> single PR.

## Goal

Replace **Avro** with **Protobuf** on every Kafka topic, **incrementally and reversibly**, then
**remove the Confluent Schema Registry entirely** as the final step — returning the platform to a
single IDL (Protobuf) for both the query RPCs and the events.

## Strategy: incremental dual-serde, per topic

Migrate topics in the existing strangler order:
`socket.outbound` → `matchmaking.*` → `match.*` → `analysis.*` → `user.events`(+ `cheat.events`).

For **each** topic, four steps:

1. **Add the Protobuf schema** in `maichess-api-contracts` (under `protos/events/...`). Map the
   common envelope (`event_id`, `event_type`, `aggregate_id`, `sequence`, `occurred_at`,
   `correlation_id`, `causation_id`, `producer`, `payload`) to a proto envelope message with a
   `oneof payload`. Publish via the standard `v*` tag handoff; bump consumers.
2. **Consumers read both** Avro and Protobuf (discriminate via the serde magic byte / registry id
   during transition).
3. **Producers switch** to Protobuf once every consumer of that topic reads both.
4. **Retire** the topic's Avro schema and the dual-read path.

Safe and reversible at every step; no synchronized multi-language cutover. Transitional dual-path
code is the accepted cost. Topic names/keys/partitions are **unchanged** — this is a payload re-
encoding, not a topology change.

## Schema Registry: keep, then remove (two phases)

- **Phase A (during migration):** keep the registry, switch to its **Protobuf serde**, retain
  BACKWARD compatibility checks. The registry mediates the dual format and keeps evolution safe.
- **Phase B (final, separate step — only after every topic is Protobuf-only):** **remove the
  Schema Registry** from the stack. Producers/consumers move to **raw Protobuf bytes**; schemas
  live solely in `maichess-api-contracts`, generated per language. This is the deliberate end
  state; do it last so the dual-format safety net stays up until the final topic converts.

## Per-language serdes

- **C# services** (`analysis`, `database`, `match-maker`, `match-manager`, `user`, `bot-arena`,
  `anticheat`, `search`): `Google.Protobuf` + Confluent Protobuf serde (Phase A) → raw proto
  deserialize (Phase B).
- **Scala** (`engine`, `move-validator`): zio-kafka + Protobuf serde (ScalaPB types) replacing
  avro4s/Avro.
- **Node** (`socket`, and any other TS consumer): ts-proto/`protobufjs` types replacing the Avro
  serde.

## Deployment (required)

- No infra to add. The **final** step **removes** the `schema-registry` Helm component
  (`maichess-deploy/helm/.../schema-registry.yaml`) and its config/env wiring from every service
  — only after Phase B. Until then it stays.
- Update each service's Kafka client config (serde + registry URL) as it migrates; remove the
  registry URL in Phase B.

## Tests (mandatory)

- Per topic: a consumer round-trips both an Avro-encoded and a Protobuf-encoded message during the
  dual-read window; a Protobuf producer's output deserializes correctly downstream.
- Envelope mapping: every envelope field + each `payload` variant round-trips through the proto
  schema.
- Phase B: raw-proto encode/decode without the registry.
- Keep each service's coverage rules; mirror Stryker/Stryker4s exclusions; run with coverage.
- Scala services: run `sbt stryker` per their config to audit the new serde paths.

## Verify (per topic, then global)

1. During the window: messages produced as Avro **and** Protobuf are both consumed correctly.
2. After producer switch: only Protobuf on the wire; consumers unaffected; ordering/idempotency/
   envelope semantics unchanged.
3. After Phase B: registry removed from Helm; full event flow (play a match, matchmaking, socket
   push, analysis, rating, cheat) works end-to-end on raw Protobuf.

## Checklist

- [ ] Proto event schemas added per topic in api-contracts (publish/bump each time).
- [ ] Dual-read consumers → producer switch → Avro retire, per topic in strangler order.
- [ ] All three language serdes migrated.
- [ ] Phase B: registry removed from stack + Helm, raw Protobuf bytes.
- [ ] Tests (dual-read, envelope round-trip, raw-proto) to coverage; Stryker/Stryker4s audited.
- [ ] [serialization-protobuf-migration.md](../../knowledge/architecture/serialization-protobuf-migration.md)
      kept in sync; the Avro section of the event-driven ADR marked superseded at the end.
