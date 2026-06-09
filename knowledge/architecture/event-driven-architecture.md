# Event-Driven Architecture (Kafka)

**Status:** Accepted — implementation in progress
**Supersedes:** the point-to-point gRPC call graph in [grpc-overview.md](../../../maichess-api-contracts/grpc-overview.md) for all *command* and *fact* flows.

## Context

maichess services communicated point-to-point: Match Manager called Move Validator,
Engine, User, and Socket directly; Match Maker called Match Manager and Socket; Analysis
called Engine, Move Validator, and Socket. This mesh couples services tightly — every new
consumer of a fact (a finished match, a move) means modifying the producer — and provides no
durable, replayable record of what happened.

## Decision

Replace the inter-service mesh with a **central Kafka event bus** and adopt **event sourcing**
for the match domain. `match.events` is the source of truth; match state is a **projection**
rebuilt from the log. Move Validator and Engine become **stream processors** in the match flow
rather than request/response endpoints.

This is the "architecturally pure" choice (chosen over a pragmatic gRPC/Kafka hybrid):
commands and facts flow exclusively through Kafka; only genuine queries and the storage adapter
remain synchronous.

### What moves to Kafka (commands + facts)

- Move submission, validation, application, bot-move request/result, draw offers, match end.
- Matchmaking enqueue/dequeue and pairing.
- Analysis session control and engine depth updates.
- All client push (`move_made`, `match_ended`, `matched`, `analysis_update`, …) via a fan-out topic.
- Rating updates on match end.

### What stays synchronous (queries + infrastructure)

- **DatabaseService** CRUD (gRPC) — the storage adapter that projections write through.
- **Read queries** — `GetMatch`, `ListMatches`, `legal-moves`, `GetUser`, `ListBots`, and the
  Move Validator SAN lookups used by Analysis (`ValidateMoveSan`, `GetLegalMovesSan`,
  `ConvertSequenceToSan`).
- **Auth** — login/register/refresh/logout (REST), local JWT verification, and
  `Auth.ValidateToken` (gRPC) for socket connect / revocation.
- **user-db** — credentials/profiles remain CRUD master data (events emitted for downstream).

A future "ultra-pure" step could eliminate the read RPCs via event-carried state transfer
(local read replicas per service); deferred because it trades the mesh for data duplication.

## Topics

Keyed by aggregate id for per-aggregate ordering (required for chess move ordering).
Convention: `<context>.<commands|events>.v<n>`.

| Topic | Key | Retention | Cleanup |
|---|---|---|---|
| `match.commands.v1` | matchId | 24h | delete |
| `match.events.v1` | matchId | infinite | delete (the log) |
| `matchmaking.commands.v1` | playerId | 1h | delete |
| `matchmaking.events.v1` | playerId | 7d | delete |
| `analysis.commands.v1` | sessionId | 1h | delete |
| `analysis.events.v1` | sessionId | 7d | delete |
| `user.events.v1` | userId | infinite | compact |
| `socket.outbound.v1` | userId / matchId | 1h | delete |

Partitions: 12 for `match.*`, 3 for the rest. Single broker in staging, 3 (RF=3) in prod.

## Serialization

**Avro** with a **Confluent Schema Registry** (`BACKWARD` compatibility). Schemas live in
`api-contracts` (see [CONTRACT_NOTES](../maichess-api-contracts) and the event schema set).
Avro is used because it has the best-supported Confluent serdes across C#, Scala, and Node, and
the event flows no longer need proto (proto is retained only for the surviving query RPCs).

Every message carries a common envelope:

| Field | Type | Purpose |
|---|---|---|
| `event_id` | uuid string | idempotency key |
| `event_type` | string | e.g. `match.MoveApplied` |
| `aggregate_id` | string | partition key (e.g. matchId) |
| `sequence` | long | per-aggregate monotonic; dedupe + gap detection |
| `occurred_at` | long (ms) | event time |
| `correlation_id` | string | one logical flow |
| `causation_id` | string | the event_id that caused this one |
| `producer` | string | emitting service |
| `payload` | union | the typed event body |

## Match flow (event-sourced)

```
POST /matches/{id}/moves  → 202 Accepted  (client awaits result via socket)
  Match Manager (command side): check participant + turn vs Redis read model
    → produce MoveSubmitted{fen, move, position_history}

Move Validator (stream processor, stateless)
  consume MoveSubmitted → pure ZIO chess logic
  produce MoveValidated{resulting_fen, game_result, position_history} | MoveRejected{reason}

Match Manager (projector)
  consume MoveValidated
    → update Redis read model; compute clocks (elapsed, decrement, +increment if ongoing)
    → produce MoveApplied
    → produce socket.outbound: move_made
    → terminal result → produce MatchEnded (+ socket match_ended, + user rating event)
    → else bot to move → produce BotMoveRequested{request_id}

Engine (stream processor)
  consume BotMoveRequested → GetBestMove logic (dedupe on request_id)
  produce BotMoveCalculated{request_id}

Match Manager
  consume BotMoveCalculated → produce MoveSubmitted (bot move) → re-enters validator loop
```

Bot and human moves share one validated path.

## Read model

Redis holds the live per-match projection (current FEN, clocks, turn, position_history blob),
rebuilt from `match.events` on startup. REST reads hit Redis; durable history is materialized
into match-db via DatabaseService. This is the CQRS read/write split.

## Client contract change

`POST /matches/{id}/moves` and `/resign` return **202 Accepted**; the authoritative result
arrives over the existing socket.io connection (`move_made` / `match_ended`). The client moves
to optimistic UI + socket confirmation. This is a deliberate REST contract change recorded here.

## Hard problems and resolutions

- **Read-after-write:** resolved by the 202 + socket model above.
- **Timeouts:** a timeout is the absence of a move; a per-match scheduled check (keyed on
  `last_move_at + remaining`) emits `MatchEnded{TIMEOUT}`. New timer component in Match Manager.
- **Idempotency:** at-least-once delivery + dedupe on `(aggregate_id, sequence)`. Validator is
  pure (safe to reprocess); Engine dedupes on `request_id` (nondeterministic). Kafka
  transactions on the validator and projector consume→produce steps give effectively-once.
- **position_history:** the validator's opaque repetition blob rides on `MoveSubmitted` /
  `MoveValidated`, keeping the validator stateless. Match Manager relays, never inspects.
- **Analysis streaming/cancel:** `StartAnalysis`/`StopAnalysis` on `analysis.commands`; engine
  streams `AnalysisDepthCompleted` to `analysis.events`; cancel via `StopAnalysis` by sessionId.
  Loses native gRPC stream backpressure (accepted trade).
- **Scala serde:** zio-kafka + Confluent Avro serde (avro4s for schema derivation).

## Observability

The topology graph is built from OTel `servicegraph` spans. Kafka client instrumentation must
stay enabled so `messaging.*` produce/consume spans (with span links) flow; the mesh then
renders as a Kafka hub — making the decoupling visible.

## Rollout (strangler)

> **Live progress / handoff:** see [event-driven-migration-status.md](../../tasks/kafka-migration-status.md)
> for what's done vs. remaining and where to pick up. As of 2026-06-09: phases 0–2 done, phase 3+ open.

0. Foundations: this ADR, Avro schemas, Helm (Kafka + registry + topic init), per-language helper.
1. `socket.outbound` (socket service consumes; producers dual-write then cut over).
2. Matchmaking.
3. Match move loop (the core: projector, validator/engine stream processors, Redis read model, timer).
4. Analysis (in maichess-mono until analysis-service is extracted).
5. User/rating events; decommission RecordMatchResult gRPC.
6. Decommission dead gRPC: EmitEvent, BroadcastMatchEvent, internal MakeMove, CreateMatch client,
   Engine.GetBestMove. Keep all query RPCs + DB CRUD + ValidateToken.
