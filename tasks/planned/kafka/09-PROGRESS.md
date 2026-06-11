# Kafka 09 — Progress log

> Running log so a fresh session can resume. Update after every meaningful step.
> Plan: decommission dead gRPC + remove the Schema Registry (raw Protobuf bytes).
> Approved plan file: `~/.claude/plans/humble-wiggling-rocket.md`.

## Status: ✅ **COMPLETE.** Phases 1–6 done. Contracts published as **v0.10.0**; every consumer bumped 0.9.0→0.10.0 (npm/socket 0.8.0→0.10.0); engine `getBestMove` removed; ALL services rebuilt + tested green; Phase 6 KB narrative docs updated. Only remaining gate: user's staging end-to-end verify.

### POST-PUBLISH (v0.10.0) — DONE
- [x] Bumped `platform-protos` 0.9.0→0.10.0 in all C# csproj (match-manager, match-maker, analysis,
      tournament-bridge, user, database ×2) + both `build.sbt` (engine, move-validator); socket
      `package.json` 0.8.0→0.10.0 (+ `npm install` relocked). (bot-arena 0.7.0 / auth 0.8.0 left as
      pre-existing skews — neither touches a removed RPC.)
- [x] Engine `getBestMove` removed: `grpc/BotsServiceImpl.scala` (impl + GetBestMove* imports),
      `Main.scala` BotsAdapter override + imports, `BotsServiceSpec.scala` getBestMove suite + import.
- [x] Rebuilt + tested ALL green: engine 213, move-validator 175, match-manager 320, match-maker 101,
      analysis 47, tournament-bridge 30, user 110, database 56 (+12 Mongo integration skipped),
      socket build + 5. No reference to any removed proto type remains.
- [x] Stale-doc cleanup: api-contracts REST narratives (socket.md, match-manager.md, match-maker.md)
      and `maichess-deploy` anticheat-service.yaml comment updated off the old `Socket.EmitEvent`/
      `BroadcastMatchEvent`/`SCHEMA_REGISTRY_URL` story.
- [x] Phase 6 KB narrative docs: serialization-protobuf-migration.md (registry removed/single-IDL),
      event-driven-architecture.md (raw bytes; engine.commands/events bot loop + topics table; final
      sync surface), kafka program README (marked COMPLETE). deployment-and-environments.md needed no
      change (it never listed the registry/topics).
- [ ] Staging end-to-end verify (USER's manual gate): play a match, matchmaking, socket push,
      analysis push, a rating update, an external game via the bridge — with NO schema-registry.

---
### (historical) Status before publish: Phases 1–4 COMPLETE + verified. Phase 5 proto edits DONE + buf-validated → BLOCKED on user contract publish (tag/push vX.Y.Z).

## Key facts established during exploration
- **Scala (engine, move-validator) already use RAW Protobuf** (`ProtobufEventSerdes.scala` =
  `parseFrom`/`toByteArray`, no registry). C# uses Confluent registry framing on the SAME
  topics → wire-incompatible TODAY on `match.events.v1`/`match.commands.v1`. Part B aligns them.
- **`user.events.v1` is still Avro on the wire** (UserCdcRelay produces GenericRecord;
  match-manager UserReplicaConsumer reads Avro). Must migrate to Protobuf as part of Part B.
- **`Engine.GetBestMove` NOT dead** — tournament-bridge uses it. User chose: rework bridge onto
  a Kafka bot-move request/reply loop (new `engine.commands.v1`/`engine.events.v1` topics),
  then remove GetBestMove.
- **`Socket.EmitEvent` NOT dead** — analysis-service uses it for client push. User chose:
  migrate analysis push onto `socket.outbound.v1`, then remove EmitEvent + socket gRPC server.
- **Genuinely dead** (after `Socket:Transport` flag removal): `BroadcastMatchEvent`,
  `Matches.MakeMove`, `Matches.ResignMatch` (gRPC overrides already gone in MatchesGrpcService).
- **`Matches.CreateMatch` STAYS** — bot-vs-bot (match-maker bot path + tournament-bridge) uses
  it for start_fen validation. Only human-path `GrpcMatchCreator` wiring is removed.

## Raw-Protobuf pattern
- C# producer: `IProducer<string, byte[]>`, `Value = msg.ToByteArray()`.
- C# consumer: `T.Parser.ParseFrom(value)`; drop registry + ProtobufDeserializer +
  ConfluentFraming + Avro arm.
- Node: `OutboundEvent.decode(value)`; drop confluent-schema-registry + Avro arm.
- Scala: no change.

## Steps
### Phase 1 — Part B
- [x] match-manager: KafkaMatchEventProducer, MatchEventProjectorConsumer, MatchCommandConsumer,
      KafkaSocketNotifier, UserReplicaConsumer → raw. Deleted ConfluentFraming, MatchCommandAvroReader,
      match.events.v1.avsc. UserReplicaProjection: Avro→proto. ProtobufEventSerdes.cs now raw
      (RawSerializer/RawDeserializer). Removed Apache.Avro + Confluent.SchemaRegistry* from csproj
      (main + test). Tests rewritten (MatchCommandReaderTests, UserReplicaProjectionTests).
      **BUILD GREEN, 286 tests pass.** (build: `set -a; . MaichessMatchManagerService/.env; set +a;
      export GITHUB_ACTOR=$GITHUB_USER` then dotnet build/test.)
- [x] match-maker: KafkaMatchCreator, KafkaMatchmakingNotifier → raw; Queue/ProtobufEventSerdes raw.
      Streaming: new ProtobufSerDes<T> (Streamiz); UserRatingTopology + UserEventReader switched to
      UserEvent proto; MatchMakerStreams dropped SchemaRegistryUrl. Deleted MatchmakingEventSerDes,
      MatchmakingAvroToProto, AvroPayload, ConfluentFraming + Avro test support
      (AvroTestData, MatchmakingAvroToProtoTests). Removed Apache.Avro + Confluent.SchemaRegistry* +
      Streamiz.*.SchemaRegistry.SerDes.Avro from csproj (main + test). **BUILD GREEN, 97 tests pass.**
      TODO: update stryker mutate excludes (deleted file globs).
- [x] analysis: AnalysisEventConsumer, KafkaAnalysisCommandSink + ProtobufEventSerdes → raw.
      Removed Confluent.SchemaRegistry* from csproj. **BUILD GREEN, 47 tests pass.**
- [x] user.events Avro→proto: CdcUserEventMapper now builds UserEvent proto (static class);
      UserCdcRelay produces raw UserEvent; MatchEndedConsumer (task 08) raw MatchEvent; new
      Kafka/ProtobufEventSerdes.cs (raw). Deleted user.events.v1.avsc + Apache.Avro +
      Confluent.SchemaRegistry* + EmbeddedResource from csproj. Tests rewritten to assert on proto.
      **BUILD GREEN, 110 tests pass.**
- [x] socket-service: consumer.ts + protobuf-serde.ts raw (dropped Confluent framing + readSchemaId);
      outbound-decode.ts dropped Avro half (fromAvro/AvroOutboundEnvelope). Removed
      @kafkajs/confluent-schema-registry from package.json. Tests rewritten. **tsc clean, 5 tests pass.**
      TODO: run `npm install` to drop the dep from package-lock.json.
- [x] contracts: deleted events/v1/*.avsc (6) + empty events/v1 dir; rewrote events/README.md
      (Avro→Protobuf, raw bytes, points to protos/events/v1/).
- [x] maichess-deploy: deleted schema-registry.yaml; removed SCHEMA_REGISTRY_URL from kafkaEnv helper;
      removed schemaRegistry values block; removed kafka-ui SCHEMAREGISTRY env; fixed stale comments
      (values.yaml, values-staging.yaml, user-service.yaml). `helm template -f values-staging.yaml` OK.
- [ ] Scala: verify sbt compile (no change — engine/move-validator already raw). DEFERRED to Phase 3/4.
- [x] Build/test all touched C#/Node services GREEN (mm 286, maker 97, user 110, analysis 47, socket 5).

### PHASE 1 (Part B) COMPLETE for C#/Node + contracts + deploy. Scala unchanged (already raw).

### Phase 2 — analysis push → socket.outbound.v1  ✅ DONE
- [x] New `Services/ISocketPushSink.cs` seam (PushAnalysisUpdate/Complete/Error). New
      `Kafka/KafkaSocketPushSink.cs` ([ExcludeFromCodeCoverage]) produces OutboundEvent with
      SocketPush{TargetUserId,event_name,payload_json} to socket.outbound.v1 (raw Protobuf serde),
      payload JSON field names identical to the old gRPC Struct (session_id/depth/lines/final_depth/
      message). AnalysisSessionService: ctor takes ISocketPushSink (was SocketGrpc.SocketClient); the
      3 Emit* methods are now thin pass-throughs to the sink. Program.cs: dropped Services:SocketService
      url + SocketClient registration; always registers ISocketPushSink→KafkaSocketPushSink.
      **BUILD GREEN, 47 tests pass.** EmitEvent now has NO caller (precondition for Phase 5).
      Phase 6 doc TODO: analysis CLAUDE.md (drop Services:SocketService env row + socket.proto client
      bullet), CONTRACT_NOTES.

### Phase 3 — tournament-bridge → Kafka bot loop  ✅ DONE (no publish needed)
- [x] **Envelope decision (no contract change):** reuse the existing `MatchEvent` envelope
      (already carries BotMoveRequested/BotMoveCalculated in its oneof) on DEDICATED topics
      engine.commands.v1 / engine.events.v1. Avoids a publish (MatchEvent is already in
      Maichess.PlatformProtos) AND keeps external requests off match.events.v1 (no phantom
      matches). A dedicated engine_commands/engine_events proto is a possible Phase-5 purity
      refinement but is NOT required for correctness — would force a re-code + publish for no
      functional gain, so left out unless the user wants it.
- [x] engine: new `kafka/EngineCommandStream.scala` (twin of EngineStream; CommandTopic=
      engine.commands.v1 input, EventTopic=engine.events.v1 output, GroupId=engine-commands;
      reuses the pure BotMoveProcessor unchanged). Wired into Main.scala kafkaWork (3-way zipPar).
      Added to build.sbt coverageExcludedFiles + stryker4s.conf mutate excludes (live-Kafka shell).
      **sbt compile + 217 tests GREEN (coverage gate satisfied).**
- [x] tournament-bridge: new `Services/IEngineMoveSource.cs` seam; `Kafka/PendingBotMoves.cs`
      (pure request_id→TCS correlation registry, UNIT-TESTED, 4 new tests); `Kafka/ProtobufEventSerdes.cs`
      (raw); `Kafka/KafkaEngineMoveSource.cs` ([Excl] producer: MatchEvent{BotMoveRequested,
      request_id}→engine.commands.v1, await correlated reply w/ 30s timeout); `Kafka/EngineEventConsumer.cs`
      ([Excl] BackgroundService consuming engine.events.v1→PendingBotMoves.Complete). Orchestrator:
      ctor takes IEngineMoveSource (dropped Bots.BotsClient); all 3 GetBestMoveAsync calls replaced.
      Program.cs: registers PendingBotMoves + IEngineMoveSource→KafkaEngineMoveSource +
      EngineEventConsumer hosted service; KEEPS Bots.BotsClient (TournamentEndpoints ListBots).
      csproj: +Confluent.Kafka 2.6.1. **BUILD GREEN, 30 tests pass.** GetBestMove now has NO bridge
      caller (precondition for Phase 5).
- [x] deploy: added engine.commands.v1 + engine.events.v1 topics (1h delete) to values.yaml
      kafka.topics. **FOUND + FIXED latent gap:** engine-service.yaml never set KAFKA_ENABLED
      (only move-validator did), so the engine's streams (native bot path task 06, analysis task 07,
      AND the new command loop) would never start in staging. Added the same
      `ternary … KAFKA_ENABLED=true … .Values.kafka.enabled` extraEnv. `helm template -f
      values-staging.yaml` renders; engine deployment now carries KAFKA_ENABLED; both topics render.

### Phase 4 — Part A code-side removal  ✅ DONE (except engine GetBestMove — see note)
- [x] match-manager: deleted `Events/SocketNotifier.cs`; Program.cs always
      `ISocketBroadcaster→KafkaSocketNotifier` (dropped Socket:Transport branch, SocketClient
      registration, Services:SocketService url, `using SocketSvc`); ISocketBroadcaster comment
      updated. Test `Support/MatchServiceContext.cs` now injects `Substitute.For<ISocketBroadcaster>()`.
      **BUILD GREEN, 286 tests pass.**
- [x] match-maker: deleted `Queue/SocketNotifier.cs` + `Queue/GrpcMatchCreator.cs` +
      test `GrpcMatchCreatorTests.cs`. Program.cs always Kafka (dropped Socket:Transport branch +
      SocketClient + Services:SocketService); IMatchmakingNotifier/IMatchCreator comments updated;
      KEEPS Matches.MatchesClient (bot-vs-bot CreateMatch). Test contexts (Queueing/Matching) inject
      `Substitute.For<IMatchmakingNotifier>()`. **BUILD GREEN, 89 tests pass** (was 97; −8 deleted
      GrpcMatchCreator tests).
- [x] socket-service: deleted `src/grpc/server.ts` (Socket EmitEvent/BroadcastMatchEvent gRPC
      server) + its bootstrap in index.ts. KEPT `src/grpc/auth-client.ts` (Auth.ValidateToken) +
      the socket.outbound.v1 consumer (emitToUser/broadcastToMatch now have only the consumer as
      caller). **tsc clean, 5 tests pass.** GRPC_PORT env now unused (Phase 6 doc).
- [~] **engine: GetBestMove impl removal DEFERRED to Phase 5 (post-publish).** The generated
      ScalaPB `BotsGrpc.Bots` trait makes `getBestMove` an abstract member, so the BotsAdapter in
      Main.scala MUST implement it until the proto drops the RPC. Cannot remove without the
      regenerated package. Do it right after the publish handoff, alongside the consumer version bump.
- [x] deploy: removed both `Socket__Transport` blocks + stale "Phase 3" comment from match-manager;
      removed `Services__SocketService` from match-manager / match-maker / analysis env; removed the
      dead 50051 gRPC port (containerPort + service) from socket-service.yaml. `helm template`
      renders for both prod and staging value sets; no `Socket__Transport` / `socket-service:50051`
      refs remain.

### Phase 5 — contract RPC removal → PUBLISH HANDOFF  ⏸️ PROTO EDITS DONE — BLOCKED on user publish
- [x] socket.proto → version-history stub (removed `Socket` service + EmitEvent/BroadcastMatchEvent
      + 4 messages; kept package + added csharp_namespace/java options for parity with analysis.proto).
- [x] bots.proto → removed `GetBestMove` RPC + GetBestMoveRequest/Response (kept ListBots, AnalyzePosition).
- [x] matches.proto → removed `MakeMove` + `ResignMatch` RPCs + their 4 messages; updated stale
      `Socket.BroadcastMatchEvent` comments to `socket.outbound.v1`.
- [x] **Event schemas: NO new engine.commands/engine.events proto added** — Phase 3 reused the existing
      `MatchEvent` envelope on dedicated topics, so nothing to add. (Dedicated envelope = optional future
      purity refinement; would need a re-code + publish, not done.)
- [x] `buf lint` + `buf build` pass (lint warnings are all PRE-EXISTING repo-wide conventions:
      package-dir layout, `*Service` suffix — my 3 files add none new). `buf breaking --against main`
      reports EXACTLY the 14 intended deletions + 3 additive socket.proto file-options. (buf via
      `npx @bufbuild/buf@latest`; not installed globally.)
- [x] CONTRACT_NOTES.md publish-handoff section written in all 5 touched services (engine,
      match-manager, socket, analysis; NEW file for tournament-bridge).
- [x] **Verified blast radius:** the ONLY remaining code references to a removed proto type are the
      engine's `GetBestMove` (BotsServiceImpl.scala + Main.scala BotsAdapter + BotsServiceSpec) — the
      deferred Phase-4 item. Every other service is clean → a version bump alone keeps them green.
- [ ] 🚧 **HARD STOP — USER ACTION REQUIRED:** commit the api-contracts changes (the 3 protos +
      the Phase-1 `.avsc` deletions), tag `vX.Y.Z`, push to publish `Maichess.PlatformProtos` /
      `@maichess/platform-protos` / Scala coord. A fresh agent shell CANNOT restore an unpublished
      package, so the consumer rebuilds below cannot run until this is done.
- [ ] POST-PUBLISH (agent, after user confirms): bump the package version in EVERY consumer
      (`*.csproj`, `build.sbt`, `package.json`); remove engine `getBestMove`
      (BotsServiceImpl.scala + Main.scala BotsAdapter + GetBestMove imports + BotsServiceSpec cases);
      final build/test ALL services incl. engine `sbt compile`/test.

### Phase 6 — docs + verify  🔶 PARTIALLY DONE (unblocked parts)
- [x] Per-service CLAUDE.md factual corrections: analysis (dropped SCHEMA_REGISTRY_URL +
      Services:SocketService env rows + socket.proto client bullet; push via ISocketPushSink),
      socket-service (role/contracts/stack/architecture/env → consumer not gRPC server; GRPC_PORT
      removed), engine (RPCs now ListBots/AnalyzePosition; GetBestMove removal note), match-manager
      (socket broadcasting via Kafka), match-maker (creation/notify always Kafka, "Avro"→raw Protobuf),
      tournament-bridge (GetBestMove→Kafka loop).
- [x] README fixes: user-service + match-maker dropped SCHEMA_REGISTRY_URL.
- [x] CONTRACT_NOTES.md publish-handoff in all 5 touched services (see Phase 5).
- [x] **Grep gate run (workspace, excl. generated/vendored):** no `.avsc`; no SCHEMA_REGISTRY_URL in
      code/deploy (only the new "it's removed" doc notes); EmitEvent/BroadcastMatchEvent only in
      explanatory comments; GetBestMove only in engine (pending publish) + the bridge's new
      IEngineMoveSource.GetBestMoveAsync seam name (not a gRPC call).
- [ ] BLOCKED/DEFERRED to post-publish finalization: KB narrative docs
      (serialization-protobuf-migration.md = registry removed/single-IDL; event-driven-architecture.md
      = final sync surface + engine.commands/events bridge loop; deployment-and-environments.md =
      schema-registry gone, engine.* topics, engine KAFKA_ENABLED; program README mark complete) +
      staging end-to-end verify (user's step).

## WHERE WE ARE (resume point)
**Phase 1 (Part B: Schema Registry → raw Protobuf) is DONE and verified.** All C#/Node services,
contracts (.avsc deleted + events/README rewritten), and maichess-deploy (schema-registry removed)
are on raw Protobuf. Every touched C#/Node service builds green with passing tests. The C# side now
matches the Scala side (which was already raw), so `match.events.v1`/`match.commands.v1` interoperate.

Build creds for any C# service: `set -a; . <repo>/services/maichess-match-manager-service/MaichessMatchManagerService/.env; set +a; export GITHUB_ACTOR=$GITHUB_USER` (the match-manager .env has GITHUB_TOKEN/GITHUB_USER; most service .envs do not).

### Remaining (Phases 2–6) — next session(s)
- **Phase 2 — analysis push → socket.outbound.v1.** Replace the 3 `socketClient.EmitEventAsync`
  calls in `analysis-service Services/AnalysisSessionService.cs` (analysis_update/complete/error,
  all user-targeted) with producing an `OutboundEvent` to `socket.outbound.v1` (payload as
  `payload_json`, mirror match-manager `KafkaSocketNotifier`). Introduce an `ISocketPushSink` seam;
  remove `SocketGrpc.SocketClient` from Program.cs. AnalysisSessionService is tested via Reqnroll
  step defs — find where it's constructed and inject a fake sink. This removes the last `EmitEvent`
  caller (precondition for removing the RPC in Phase 5).
- **Phase 3 — tournament-bridge → Kafka bot loop (LARGEST; needs a contract publish).** New
  `engine.commands.v1` (BotMoveRequested) + `engine.events.v1` (BotMoveCalculated) topics (reuse the
  match_events messages). Add topics to maichess-deploy `kafka.topics`. Engine: add an
  `EngineCommandStream` (mirror `EngineStream`) consuming engine.commands → `BotMoveProcessor` →
  engine.events. tournament-bridge `TournamentOrchestrator`/`GameDriver`: replace the 3
  `engineClient.GetBestMoveAsync` calls with produce-BotMoveRequested / await-BotMoveCalculated
  correlated by request_id (TaskCompletionSource registry + timeout). Removes the last GetBestMove
  caller (precondition for Phase 5). Do NOT route through `match.events.v1` — the match projector
  consumes all of it and would create phantom live matches.
- **Phase 4 — Part A code-side removal (mechanical, no publish).**
  - match-manager: remove `Events/SocketNotifier.cs` + the `Socket:Transport` branch + SocketClient
    wiring in Program.cs (always KafkaSocketNotifier). Verify/remove dead MatchService.MakeMoveAsync/
    ResignMatchAsync only if REST no longer calls them (it does today — `MatchesEndpoints.cs:225/255`;
    confirm task 06 intent before deleting; the *proto RPC* removal is what Phase 5 does).
  - match-maker: remove `Queue/SocketNotifier.cs` + `Queue/GrpcMatchCreator.cs` + the
    `Socket:Transport` branch (always Kafka). KEEP `Matches.MatchesClient` (bot-vs-bot CreateMatch).
  - socket-service: remove the Socket gRPC server (after Phase 2 so EmitEvent has no caller); keep
    the Auth.ValidateToken client.
  - engine: remove GetBestMove from `grpc/BotsServiceImpl.scala` + `Main.scala` (after Phase 3); keep
    ListBots. Leave AnalyzePosition (out of scope; still flag-gated).
  - Remove `Socket:Transport` from maichess-deploy values/templates.
- **Phase 5 — contract RPC removal → PUBLISH HANDOFF.** Edit protos: socket.proto (remove Socket
  service → stub like analysis.proto), bots.proto (remove GetBestMove + msgs), matches.proto (remove
  MakeMove + ResignMatch + msgs), add engine.commands/engine.events schemas (Phase 3). `buf lint` +
  `buf breaking`. STOP for user tag/push vX.Y.Z; then bump Maichess.PlatformProtos in EVERY consumer
  (*.csproj, build.sbt, package.json) and final build/test (incl. Scala `sbt compile`).
- **Phase 6 — docs + verify.** Update serialization-protobuf-migration (registry removed),
  event-driven-architecture (final sync surface + bridge bot loop), deployment-and-environments,
  program README (mark complete), per-service CLAUDE/CONTRACT_NOTES (analysis SCHEMA_REGISTRY_URL env
  row, socket gRPC removal, engine GetBestMove). Staging end-to-end + grep gate.

## Blockers
- **🚧 ACTIVE — Phase 5 contract publish handoff.** The api-contracts changes (socket.proto stub,
  bots.proto −GetBestMove, matches.proto −MakeMove/−ResignMatch, + the Phase-1 `.avsc` deletions) are
  edited and buf-validated but UNCOMMITTED on branch `dev`. The USER must commit, tag `vX.Y.Z`, push
  to publish `Maichess.PlatformProtos` / `@maichess/platform-protos` / Scala coord. A fresh agent
  shell cannot restore an unpublished package, so the consumer version bumps + the engine
  `getBestMove` removal + the final all-services rebuild cannot proceed until then.
- Scala engine compiles GREEN today on the CURRENT package (0.7.0) because BotsServiceImpl/Main still
  implement getBestMove. That removal happens right after the publish (see engine CONTRACT_NOTES.md).

## Post-publish checklist (agent resumes here once user confirms tag/push)
1. Bump platform-protos version: `*.csproj` (match-manager, match-maker, analysis, tournament-bridge,
   user, auth, socket via package.json), `build.sbt` (engine, move-validator), socket-service
   `package.json`. Use the new vX.Y.Z.
2. Engine: delete `getBestMove` from `grpc/BotsServiceImpl.scala` + `Main.scala` BotsAdapter + the
   `GetBestMove*` imports; delete the getBestMove cases in `BotsServiceSpec.scala`.
3. Rebuild/test ALL: match-manager (286), match-maker (89), analysis (47), tournament-bridge (30),
   socket-service (5), user (110), engine (`sbt compile`/test). Confirm no reference to a removed type.
4. Finish Phase 6 KB narrative docs + program README; user runs staging end-to-end verify.
