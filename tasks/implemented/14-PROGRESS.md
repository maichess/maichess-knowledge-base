# 14 — Anti-Cheat Service — Progress log

Working log for `14-anticheat-service.md`. Records design decisions, what is done, and
the exact remaining steps so another session can continue without re-deriving.

## Exploration findings (done — do not re-derive)

- Repo `services/maichess-anticheat-service/` exists but is **empty** (README only).
- Contracts already hold `protos/events/v1/cheat_events.proto` (CheatEvent with
  PlayerFlagged / PlayerUnflagged / LiveSuspicionRaised, keyed by userId) — written in
  Kafka task 01, **published in PlatformProtos 0.6.0** (all C# services are on 0.6.0).
  The Avro stub `events/v1/cheat.events.v1.avsc` was never on the wire (topic not live);
  the topic is **born Protobuf** — the avsc stub is deleted as part of this task.
- `match.events.v1` is Protobuf on the wire (Kafka tasks 04–06); serde pattern =
  match-manager `Events/ProtobufEventSerdes.cs` (Confluent ProtobufSerializer /
  ProtobufDeserializer + SyncOverAsync). Consume with that.
- **Pre-move gap:** the client implements premoves purely client-side
  (`lib/hooks/usePremove.ts`; MatchClient submits the queued move via the normal
  `makeMove` as soon as it is the player's turn). No marker reaches the server, and
  `MoveSubmitted`/`MoveApplied` have **no premove field** → contract change required
  (additive proto fields + REST `premove` flag on POST /moves). Needs **v0.7.0 publish**.
- Per-ply timing is NOT in the durable match doc (`MatchDocument` has Moves/FenHistory but
  no per-move timestamps). Timing comes from the `match.events` stream:
  `MoveApplied.applied_at_ms` deltas (envelope `occurred_at` of `MatchCreated` seeds ply 0).
  → the anticheat service accumulates per-match timing state from the stream (shared
  between iteration 2 live scoring and the iteration 1 post-game pass); the match-db fetch
  supplies the authoritative move list/FENs for engine correlation.
- User read models that must gain `flagged`:
  - match-manager Redis user replica: `Kafka/UserReplicaConsumer.cs` (Avro GenericRecord)
    → pure `Kafka/UserReplicaProjection.cs` → `Data/RedisUserReplica.cs` (`user:{id}` hash).
    Add a parallel cheat.events consumer (Protobuf) + pure projection.
  - match-maker KTable value `Streaming/UserRatingState.cs` already has `Flagged`
    (defaults false, comment says "populated by a later stage — cheat.events").
    `MatchingService.SelectPairAsync` currently skips flagged players in the *skill* path
    only; FIFO path (`DequeueOldestPairAsync`) is blind. Toggle semantics must filter both.
- Dev gate: JWT has no dev_mode claim → anticheat REST resolves the caller via
  user-service `Users.GetUser` (proto has `dev_mode`, field 7) and 403s non-dev.
- anticheat-db = another database-service instance (Mongo). database-service
  `Program.cs` switches migrations on `Database:Migrations` domain names; Mongo has
  `match`/`arena` → add `AnticheatMongoMigration` (domain `anticheat`, collections
  `cases` + `audit`, indexes on user_id/status).
- Engine correlation RPC: `protos/engine-service/v1/bots.proto` `AnalyzePosition`
  (streams AnalysisUpdate{depth, lines[rank, evaluation_cp, moves]}); take the final
  (deepest) update. Also `GetBestMove`.
- Helm scaffolds exist & are gated off: `templates/anticheat-service.yaml`,
  `anticheat-db.yaml`; topic `cheat.events.v1` (compact) already in values.yaml.
- Template service for layout/analyzer-strictness: **maichess-search-service** (flat
  layout, net10.0, TreatWarningsAsErrors + AnalysisMode All + StyleCop, one type per
  file, xUnit facts, coverlet 100% on non-excluded, stryker mirroring exclusions).

## Design decisions (committed)

1. **Wire:** cheat.events.v1 is Protobuf from day one (CheatEvent from PlatformProtos
   0.6.0); the avsc stub is removed. Consumers of cheat.events MUST ignore
   `LiveSuspicionRaised` (advisory) — only PlayerFlagged/PlayerUnflagged touch `flagged`.
2. **Premove contract (v0.7.0):** `MoveSubmitted.premove = 5`, `MoveApplied.premove = 8`
   (additive); REST POST /matches/{id}/moves gains optional `premove` (default false).
   Client-asserted; it only *exempts a ply from timing analysis* and downweights it in
   correlation — it never reduces engine-correlation evidence to zero, so marking
   everything premove cannot hide cheating (documented in rest specs + ADR).
3. **anticheat-service shape** (flat, namespace `MaichessAnticheatService`):
   - `Detection/` pure + fully tested: `PlyObservation` (ply, userId, moveUci, fenBefore,
     thinkTimeMs, premove), `EngineEvaluation` (top lines), `EngineCorrelationDetector`
     (weighted top-1/2/3 match rate over non-book plies, premove plies weight 0.5,
     baseline-normalised), `StatisticalDetector` (think-time coefficient-of-variation over
     non-premove non-book plies w/ min samples; rating-vs-performance bump), `CombinedScorer`
     (0.7*corr + 0.3*stat per game; case score = mean over last K games, flag at
     threshold with >= MinGames), `LiveScorer` (iteration 2: incremental fold of
     PlyObservations per (match,player); timing-only advisory signal, once per
     match/player). All thresholds in `DetectionOptions` (bound from config).
   - `Stream/` pure + tested: `MatchStreamProjection` — folds MatchEvent (MatchCreated/
     MoveApplied/MatchEnded) into per-match `MatchTimingState` (players, per-ply
     PlyObservation list, premove bit from MoveApplied once 0.7.0 lands — until then the
     fold takes the bool from the proto field default false).
   - `Analysis/`: `GameAnalyzer` (orchestrates: IMatchReader fetch + IEngineAnalyzer per
     ply + detectors → GameVerdict; pure-ish, tested with fakes), `IEngineAnalyzer` +
     `GrpcEngineAnalyzer` (excluded; AnalyzePosition line_count=3, last update,
     SemaphoreSlim rate limit `Analysis:MaxConcurrent` default 1 + per-ply delay).
   - `Cases/`: `CaseService` (tested) over `IAnticheatStore` (+ `DatabaseAnticheatStore`
     excluded, Database gRPC to anticheat-db): collections `cases` {id, user_id, status
     open|flagged|cleared, score, games[{match_id, score, correlation, statistical,
     analyzed_at_ms, suspicious_plies}], created/updated/flagged_at_ms} and `audit`
     {id, case_id, user_id, action flagged|unflagged|live_suspicion, actor, reason, at_ms,
     score}. References match-db by match_id only — no move/FEN copies.
   - `Kafka/`: `MatchEventConsumer` (excluded shell) → routes through the pure pieces;
     MatchEnded enqueues to a bounded Channel consumed by `AnalysisWorker` (excluded
     BackgroundService, rate-limited engine load); `ICheatEventProducer` +
     `KafkaCheatEventProducer` (excluded, keyed by user_id).
   - `Rest/AnticheatEndpoints.cs` (excluded): GET /anticheat/cases?status=,
     GET /anticheat/cases/{id}, POST /anticheat/cases/{id}/unflag {reason} — Bearer JWT +
     dev gate via `IDevGate` (`UserServiceDevGate` excluded, Users.GetUser dev_mode).
   - match-db reads via `IMatchReader`/`DatabaseMatchReader` (excluded, Database gRPC
     Get on `matches`).
4. **Iteration 1 timing source:** the per-match observations accumulated from the stream;
   if the state is cold (e.g. consumer restarted mid-game without replay) the post-game
   pass runs **correlation-only** (timing features need observations; documented).
5. **Live signal surfacing (iteration 2):** `LiveSuspicionRaised` on cheat.events.v1 +
   an `audit` entry (action live_suspicion) so the Dev overview shows it. Read models
   ignore it (tested). Authoritative flag only from the post-game pass.
6. **Matchmaking toggle:** POST /queue gains optional `allow_flagged` (default false).
   QueueEntry stores it. Admissibility for pair (A,B): (!flagged(B) || allow(A)) &&
   (!flagged(A) || allow(B)). Applied in BOTH paths: skill (filter candidates pairwise —
   compute closest admissible pair) and FIFO (oldest admissible pair via waiting list +
   DequeueSpecificPairAsync). Flag source in match-maker: in-memory `CheatFlagStore`
   fed by a from-beginning consumer of compacted cheat.events.v1 (simpler than touching
   the Streamiz topology; `UserRatingState.Flagged` comment updated accordingly).
7. **match-manager replica:** pure `Kafka/CheatFlagProjection.cs` (CheatEvent →
   `flagged` upsert; live_suspicion → null) + `Kafka/CheatFlagConsumer.cs` (excluded,
   pattern = UserReplicaConsumer but Protobuf serde) writing `user:{id}` hash.

## Status / checklist

- [x] Exploration + design (this doc).
- [x] **Phase 1 — contracts** DONE: premove proto fields (MoveSubmitted=5, MoveApplied=8;
      no collision with the concurrently merged kafka-08 bot-elo fields); rest/anticheat.md;
      rest/match-maker.md allow_flagged; rest/match-manager.md premove; cheat.events.v1.avsc
      deleted; buf build clean. → **USER ACTION: publish vNext (0.7.0)** — bundles kafka-08's
      MatchCreated/MatchEnded fields too. User was told in-session.
- [x] **Phase 2 — anticheat-service** DONE: 67 tests, **100% line/branch/method**.
      Pinned to 0.6.0; premove wire-read flip + match-manager premove production are the
      documented 0.7.0 follow-up (service CONTRACT_NOTES.md has the exact 3 steps).
      Layout: Detection/ (correlation, statistical, combined, live Welford scorer),
      Stream/ (MatchStreamProjection fold + AnticheatStreamProcessor decision core),
      Analysis/ (GameAnalyzer + GrpcEngineAnalyzer excluded), Cases/ (CaseService,
      CheatEvents builders, store seam), Data/ + Kafka/ + Rest/ excluded glue, stryker
      mirrored, CLAUDE.md/README/CONTRACT_NOTES written. Deviations recorded:
      page-local list ordering, accuracy-spike instead of rating-vs-performance,
      timing-only live scoring.
- [x] **Phase 3 — propagation** DONE.
      - match-manager: `Kafka/CheatFlagProjection.cs` (pure, 100%) + `Kafka/CheatFlagConsumer.cs`
        (excluded shell, Protobuf serde, sets `flagged` on `user:{id}` replica hash) wired in
        Program.cs after UserReplicaConsumer; stryker exclusion added; CLAUDE.md updated.
        `CheatFlagProjectionTests` (4 facts). Full suite **297 tests green, 100%** on the new
        file (overall still the 4 pre-existing baseline partials only).
      - match-maker: removed the never-populated `Flagged` from `UserRatingState`/
        `SkillEnrichedEnqueue` (KTable no longer carries it); new `Streaming/CheatFlagStore`
        (`ICheatFlagStore`, in-memory) + `CheatFlagProjection` + `CheatFlagConsumer` (excluded,
        per-run group id replays the compacted topic). Toggle: `QueueRequest.AllowFlagged`,
        `QueueEntry.AllowFlagged` (Redis `allow_flagged` field), `QueueingService.EnqueueAsync`
        +`allowFlagged`, `IQueueRepository.EnqueueAsync`/`GetWaitingPlayersAsync` carry it.
        `MatchingService` rewritten: `IsAdmissible`, `ClosestRatedPair` (now scans all pairs for
        admissibility), `OldestAdmissiblePair`, FIFO re-reads waiting list so a lost skill race
        still falls back within the tick. **116 tests green**; my new code 100% (the only
        uncovered lines are the PRE-EXISTING `MatchmakingAvroToProto` PlayersMatched branch +
        `AvroPayload` branch 22 — kafka-task-02 infra, the test Avro schema doesn't even define
        PlayersMatched; untouched by task 14).
      - **NOTE for the 0.7.0 bump:** match-manager/match-maker already on PlatformProtos 0.6.0
        which has CheatEvent, so these compiled/tested now. The 0.7.0 premove follow-up only
        affects match-manager's *move production* + anticheat's *premove read* (see service
        CONTRACT_NOTES), not flag propagation.
- [x] **Phase 4 — client** DONE (`npm run build` + `lint` clean for my files; the only
      lint errors are PRE-EXISTING in useAnalysisSession/useAnalysisBoardInput/PlayerCard/
      auth-logout — not mine). Premove: `useMatch.makeMove(uci, premove=false)` sends
      `premove` in the body; `MatchClient` premove-firing effect calls `makeMove(uci, true)`.
      Toggle: `QueueRequest.allow_flagged`, `app/play/page.tsx` checkbox (human only) →
      human request. Dev: `lib/models/anticheat.ts`, `useAnticheat` hook, `AnticheatPanel`
      (list+detail+unflag via window.prompt reason), `app/dev/anticheat/page.tsx`
      (requireDevUser), 3 proxy routes under `app/api/anticheat/` (ANTICHEAT_SERVICE_URL),
      route constant + dev-index card. helm client env gains `ANTICHEAT_SERVICE_URL`.
- [x] **Phase 5 — helm + db** DONE. `AnticheatMongoMigration` (domain `anticheat`,
      collections cases/audit, indexes user_id/status/case_id) added + registered in
      database-service Program.cs (builds). anticheat-db template already had
      `Database__Migrations=anticheat`. anticheat-service.yaml: added `extraEnv` (Jwt__Key
      secret + conditional `Kafka__Enabled` when kafka.enabled); values.yaml env keys fixed
      to the real config (`Services__AnticheatDatabase/MatchDatabase/EngineService/UserService`).
      Service Program.cs now reads `KAFKA_BOOTSTRAP`/`SCHEMA_REGISTRY_URL` env (platform macro),
      gated on `Kafka:Enabled`. `cheat.events.v1` topic already in values (compact, retention -1).
      `helm template --show-only` renders both anticheat templates cleanly (helm v4.1.4 present;
      render scoped to avoid the pre-existing match-maker-service.yaml nested-comment issue).
- [x] **Phase 6 — docs/memory** DONE: `knowledge/services/anticheat-service.md` gained an
      "Implementation decisions (task 14 — DONE)" section; service `CONTRACT_NOTES.md` has the
      0.7.0 premove follow-up (the 3-step flip); memory `project_anticheat.md` + MEMORY.md index.

## FINAL STATUS — all phases complete; one user action remains

Every phase done and verified. Test tallies (all green, my new code 100%):
anticheat **67**, match-manager **297**, match-maker **116**, database-service **56** (+12
skipped, pre-existing). Client `npm run build` + `lint` clean (pre-existing lint errors only).
helm renders. **The single remaining action is the user publishing api-contracts vNext
(0.7.0)** for the additive premove fields, then the documented one-line premove-read flip +
match-manager premove production (service CONTRACT_NOTES). Everything else is shippable as-is
(0.6.0 already carries CheatEvent, so flag propagation + the whole detector work today).

## Pending user actions (collect here)

1. Publish api-contracts **v0.7.0** after Phase 1 (additive premove fields). Fresh
   publishes can't be restored from Claude's shell (401) — after publish, bump
   PlatformProtos 0.6.0→0.7.0 in match-manager (+anticheat if flipping the premove read)
   and run `dotnet test`.
2. Run `helm lint`/`template` after Phase 5 if helm still trips on the pre-existing
   match-maker-service.yaml nested-comment issue.
