# Event-Driven Migration — Status & Remaining Work

**As of:** 2026-06-09
**Companion to:** [event-driven-architecture.md](event-driven-architecture.md) (the ADR — read it first for the
target design, topics, envelope, and the match-flow diagram).

This is a handoff/progress log for the Kafka migration so the work can be picked up mid-flight.
The rollout follows the 6-phase strangler plan at the end of the ADR.

## Snapshot

| Phase | Scope | Status |
|---|---|---|
| 0 | Foundations: ADR, Avro schemas, Helm (Kafka + registry + topics), CONTRACT_NOTES | **Done** |
| 1 | `socket.outbound.v1` fan-out | **Code complete; match-manager transport reverted to gRPC (see caveat)** |
| 2 | Matchmaking facts/pushes | **Done** |
| 3 | Match move loop (the core) | **Not started**, except an idle CreateMatch consumer |
| 4 | Analysis over Kafka | **Not started** |
| 5 | User/rating events | **Not started** |
| 6 | Decommission dead gRPC | **Not started** |

**Only the socket fan-out and matchmaking facts ride Kafka today. The entire move pipeline is still
synchronous gRPC** (match-manager → move-validator / engine / user-service).

## Done

- **Phase 0** — ADR; Avro schemas in `maichess-api-contracts/events/v1/*.avsc`; Helm `kafka`,
  `schema-registry`, `kafka-topics-init` (creates `match.commands.v1`, `match.events.v1`,
  `matchmaking.*`, `analysis.*`, `user.events.v1`, `socket.outbound.v1`); per-service CONTRACT_NOTES.
  Kafka is `enabled` in staging only (`values-staging.yaml`); prod has `kafka.enabled: false`.
- **Phase 1** — socket-service consumes `socket.outbound.v1` (`src/kafka/consumer.ts`) and fans out over
  socket.io (room `match:<id>` for `target_match_id`, user socket for `target_user_id`). Producers:
  match-manager `Events/KafkaSocketNotifier.cs`, match-maker `Queue/KafkaMatchmakingNotifier.cs`.
  Transport chosen by the `Socket:Transport` config flag (code default `kafka`).
- **Phase 2** — match-maker publishes `matched` → `socket.outbound.v1` (user-targeted) and
  `PlayersMatched` → `matchmaking.events.v1` (`Queue/IMatchmakingNotifier.cs`). Matchmaking status is
  still **polled** by the client (`GET /queue/{token}/status`), so the `matched` socket push is not yet
  load-bearing.

## ⚠️ Current caveat (Phase 1, match-manager)

Live moves were not appearing in staging (only after a page reload). Root cause: the synchronous move
flow still persists to match-db (so reload works), but the real-time push had been cut over to the Kafka
`socket.outbound.v1` hop, which was failing **silently** at runtime (producer publish and consumer decode
are both fire-and-forget). The producer/consumer/client code is all correct, so the failure is runtime,
not logic.

**Interim fix applied:** `Socket__Transport: grpc` set for match-manager in
`maichess-deploy/helm/maichess/values.yaml` → match-manager uses the proven `SocketNotifier`
(`Socket.BroadcastMatchEvent` gRPC; socket-service still serves that handler). Set in **base** values so
it also protects prod (where Kafka is off) from the same silent drop.

**To finish Phase 1 properly:** verify the socket.outbound hop end-to-end in staging
(`kubectl logs` socket-service + match-manager, grep `outbound|consumer|decode` — distinguish
consumer-disabled / decode-error / broker-unreachable), then flip `Socket__Transport` back to `kafka`.
Ideally do this together with Phase 3 so the fan-out and the move loop are validated at once.

## Remaining work

### Phase 3 — Match move loop (the core)

**CreateMatch over Kafka — half done:**
- DONE (idle): match-manager `Events/MatchCommandConsumer.cs` consumes `match.commands.v1`
  `CreateMatchCommand` and creates the match with the caller-minted id
  (`DatabaseService.Insert` already honors a supplied non-empty id). Registered in `Program.cs`.
- TODO: match-maker still calls gRPC `Matches.CreateMatch` (`Queue/MatchingService.cs`,
  `Queue/QueueingService.cs`, 3 sites). It must mint the matchId and publish `CreateMatchCommand` via a
  new `IMatchCommandPublisher`, and its Reqnroll feature tests reworked from "gRPC CreateMatch" to
  "CreateMatchCommand published". **bot-vs-bot stays gRPC** (synchronous start_fen validation).

**The move loop itself — not started.** Today `MatchService.MakeMoveAsync` / `ProcessBotMoveAsync`
validate via gRPC `MovesClient.ValidateMove`, request bot moves via gRPC `BotsClient.GetBestMove`, write
to Mongo, then broadcast — all synchronous. `match.events.v1.avsc` is embedded but **has no producer**.
Build out (see the ADR match-flow diagram for the exact event sequence):
- match-manager command side: on `POST /moves`, check participant+turn against the read model and produce
  `MoveSubmitted` (carry the opaque `position_history` blob); return **202 Accepted**.
- move-validator: become a **stream processor** — consume `MoveSubmitted`, run pure ZIO chess logic,
  produce `MoveValidated` / `MoveRejected`. **No Kafka code exists yet** (CONTRACT_NOTES only); add
  zio-kafka + Confluent Avro serde (avro4s for schema derivation).
- match-manager **projector**: consume `MoveValidated` → update read model + clocks → produce
  `MoveApplied` → `socket.outbound` `move_made`; terminal → `MatchEnded` (+ socket `match_ended` + user
  rating event); else bot to move → `BotMoveRequested{request_id}`.
- engine: become a **stream processor** — consume `BotMoveRequested` (dedupe on `request_id`), produce
  `BotMoveCalculated`. **No Kafka code exists yet** (CONTRACT_NOTES only).
- match-manager: consume `BotMoveCalculated` → produce `MoveSubmitted` (bot move) → re-enter the loop.

**Read model (Redis) — not implemented.** match-db (Mongo) is still the live source of truth;
match-manager has no `StackExchange.Redis` dependency (the `ConnectionStrings__Redis` env is wired in the
chart but unused). Build the live per-match projection in Redis (current FEN, clocks, turn,
`position_history` blob), rebuilt from `match.events` on startup. REST reads switch to Redis; durable
history is materialized to match-db via `DatabaseService`. This is the CQRS read/write split.

**Other Phase-3 items:**
- 202 contract: `POST /moves` and `/resign` still return **200 with the full match**. The client already
  does optimistic UI + socket confirmation, but the REST handlers and contract must move to 202.
- Timeouts: `Services/TimeoutWatchdog.cs` still polls Mongo and broadcasts directly. Replace with a
  per-match scheduled check that emits `MatchEnded{TIMEOUT}`.
- Idempotency/ordering: dedupe on `(aggregate_id, sequence)`; validator is pure (safe to reprocess);
  engine dedupes on `request_id`; use Kafka transactions on the validator + projector consume→produce
  steps for effectively-once.

### Phase 4 — Analysis over Kafka
Not started. Analysis logic still lives in **maichess-mono** (analysis-service is contracts-only).
`analysis.commands.v1` / `analysis.events.v1` topics exist but are unused. Target: `StartAnalysis` /
`StopAnalysis` on `analysis.commands`; engine streams `AnalysisDepthCompleted` to `analysis.events`;
cancel by sessionId. Loses native gRPC stream backpressure (accepted in the ADR).

### Phase 5 — User / rating events
Not started. match-manager still calls user-service `RecordMatchResult` over gRPC
(`MatchService.RecordMatchResultsAsync`). `user.events.v1` exists but has no rating producer/consumer.
Emit rating updates on match end via events, then decommission the `RecordMatchResult` gRPC.

### Phase 6 — Decommission dead gRPC
Not started. Remove `Socket.EmitEvent` / `Socket.BroadcastMatchEvent` (socket-service still runs its gRPC
server), internal `MakeMove`, the `CreateMatch` client, and `Engine.GetBestMove`. **Keep** all query RPCs
(`GetMatch`, `ListMatches`, legal-moves, `GetUser`, `ListBots`, the validator SAN lookups), DatabaseService
CRUD, and `Auth.ValidateToken`.

## Conventions established (follow these)

- **Envelope** on every message: `event_id`, `event_type`, `aggregate_id` (partition key), `sequence`
  (per-aggregate monotonic), `occurred_at`, `correlation_id`, `causation_id`, `producer`, `payload` (union).
- **Serialization:** Avro + Confluent Schema Registry (`BACKWARD` compat). Canonical schemas live in
  `maichess-api-contracts/events/v1/`; services **vendor a copy** of the needed `.avsc` as an
  `EmbeddedResource` (keep byte-identical to canonical).
- **C# producers:** `Confluent.Kafka` + `Apache.Avro` `GenericRecord`, fire-and-forget publish,
  `[ExcludeFromCodeCoverage]` on the I/O glue. Per-service `.editorconfig` differs (match-maker wants
  `var` + CA1873-guarded logs; match-manager wants explicit types) — build to catch.
- **Scala:** zio-kafka + Confluent Avro serde (avro4s).
- **DatabaseService.Insert** honors a caller-supplied non-empty `id` (Mongo uses it as `_id`,
  duplicate → `AlreadyExists`; Postgres self-assigns).
- **Contract policy:** event-schema changes are contract changes → ADR + new schema in `api-contracts` +
  CONTRACT_NOTES before implementing.
- **Observability:** keep Kafka client instrumentation on so `messaging.*` spans flow (the topology graph
  renders the bus from OTel servicegraph spans).

## Deploy notes

- Staging deploys from maichess-deploy `dev` via the `update-containers.yml` workflow (helm upgrade +
  rollout restart). The `Socket__Transport` change is config-only (no image rebuild) but needs a
  **helm upgrade** — a bare `kubectl rollout restart` won't pick up the new env.
- Service repos: `dev` → `:nightly` images (staging), `main` → `:main` images (prod). Kafka is on in
  staging only.
