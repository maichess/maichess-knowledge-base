# Kafka 08 — Progress log

> Running log so a fresh session can resume. Update after every meaningful step.

## Status: DESIGN SETTLED — implementing

## Context gathered
- Task: replace match-manager → user-service `RecordMatchResult` gRPC with an event path.
  Recommended design (from the task file): **user-service consumes `MatchEnded` from
  `match.events.v1`**, runs the existing Glicko-2 update + W/L/D increment, writes Postgres only;
  the rating change flows out on `user.events.v1` via the existing CDC relay (`UserCdcRelay` →
  `CdcUserEventMapper` already maps rating-field changes to `RatingUpdated`). No second writer to
  the user store, no dual write.
- Rules to preserve: bot-vs-bot → nobody rated; external games unrated; exactly one rating
  mutation per human per finished match, idempotent on (match id, participant).
- CDC doc: `user.events.v1` is CDC-derived; do NOT produce rating events directly from
  match-manager or user-service app code.
- Glicko-2: pure module `Rating/Glicko2.cs` in user-service; gRPC request carried
  `opponent_rating`/`opponent_rd` snapshots (match-manager snapshot both sides pre-update;
  bot opponents = engine elo with RD=50, unknown bot → rating 0).
- Open question to resolve in code: does `MatchEnded` payload carry enough (participants,
  per-side outcome, bot elos, source native/external) for user-service to do the update alone?

## Steps done
- [x] Pulled knowledge-base, user-service, match-manager, api-contracts, database-service.
- [x] Read MatchEnded proto + projector emission site (match-manager)
- [x] Read user-service RecordMatchResultAsync + Kafka infra
- [x] Decide consumer design + idempotency store (see Decisions)
- [x] **api-contracts**: enriched `MatchCreated` (+white/black_bot_elo 9/10) and `MatchEnded`
      (+white/black Player 4/5, source 6, white/black_bot_elo 7/8) in
      `protos/events/v1/match_events.proto`. `npx @bufbuild/buf breaking` clean; `dotnet build`
      in dotnet/ clean (repo-wide `buf lint` failures are pre-existing style issues, untouched).
      **UNCOMMITTED — user must commit, tag vX.Y.Z (suggest v0.7.0), push.**
- [x] **database-service** (branch dev): `rated_matches TEXT NOT NULL DEFAULT '[]'` added to
      `UserPostgresMigration` (idempotent ALTER). Builds. Migrations are coverage-excluded, no
      test changes.
- [x] **user-service**: `Rating/MatchEndedFact|MatchEndedParticipant|MatchEndedStatus` DTOs +
      `UsersService.ApplyMatchEndedAsync` (trigger + idempotency; shares `ResultFields` with the
      legacy RPC path, which is unchanged in behavior). 18 new xUnit tests in
      `Tests/Rating/ApplyMatchEndedTests.cs`; suite 98/98 green; coverage 100/100/100.
- [x] CONTRACT_NOTES.md (user-service) — kafka-08 section with the publish blocker.
- [x] `dotnet stryker --mutate "**/UsersService.cs"`: first run 97.69% — 2 pre-existing
      UserFromStruct field-name mutants killed by making the HumanVsHuman test rows use
      fractional rating + non-default volatility; the remaining survivor (block-removal of the
      `catch NotFound { return null; }` body in TryGetRecordAsync) is **equivalent** (falls
      through to an injected `return default` = null). Re-run confirmed: 99.23%, only the
      equivalent mutant survives.
      NOTE Session B: add the new consumer glue file to stryker `mutate` exclusions (mirror
      `!**/Kafka/UserCdcRelay.cs`).
- [ ] **HANDOFF: user publishes contracts tag** → then Session B below.
- [ ] Bump Maichess.PlatformProtos (0.6.0 → new) in ALL consumer services (reconcile versions).
- [ ] user-service consumer glue: BackgroundService consuming match.events.v1 (Protobuf serde,
      group e.g. `user-service-rating`, config-gated like Cdc:Enabled, WARN-log decode failures,
      commit offsets after successful apply) → maps proto → MatchEndedFact. Excluded from
      coverage. Wire in Program.cs + appsettings (+ Helm env in maichess-deploy).
- [ ] match-manager enrichment: CreateMatchAsync resolves bot elos via existing BotsClient
      (ListBots) and stamps MatchCreated; LiveMatchState += Source/WhiteBotElo/BlackBotElo;
      MatchProjection.Init folds them; MatchProjector + MatchCommands fill the new MatchEnded
      fields from state. Tests to 100%.
- [ ] Verify (staging): rated human game → both ratings move once via events; replay no-op.
- [ ] Update docs: rating-glicko2.md + match-history-and-stats.md (ratings driven by MatchEnded
      events; synchronous RecordMatchResult call gone); move this task to implemented/ when done.

## Key code facts (verified)
- Task 06 already **removed** the synchronous `RecordMatchResultsAsync` from match-manager
  (commit 92821b1) — the event-loop match-end path records no ratings today. 08 closes that gap;
  the user-service RPC handler itself stays until 09.
- `MatchEnded` (match_events.proto) carries only status/end_reason/finished_at_ms — **no
  participants**. `MatchCreated` carries white/black/source but no bot elo.
- Old gRPC logic (recovered from `git show 92821b1^:.../MatchService.cs`): outcomes from status
  per color; snapshot BOTH opponents' ratings before recording either; bot opponent =
  engine-configured elo (ListBots) with fixed RD=50, unknown bot → rating 0; replica-first read
  with GetUser fallback for human opponents.
- **External matches never enter the event loop** (direct insert + SyncExternalMatch, no events)
  — but we still carry `source` in the enriched event for defense in depth.
- `user.events.v1` is still **Avro** on the wire (CDC relay `UserCdcRelay` produces
  GenericRecord; match-manager `UserReplicaConsumer` reads Avro). Untouched by 08 — the rating
  change flows out via existing CDC `RatingUpdated` mapping automatically.
- user-db is real-columns Postgres via database-service generic CRUD
  (`PostgresRecordRepository`): Insert self-assigns id; unique violation → AlreadyExists;
  migrations live in database-service `Adapters/Postgres/Migrations/UserPostgresMigration.cs`.
- match-manager `LiveMatchState` has White/Black `PlayerRef(UserId, BotId)` but **no Source, no
  bot elo** — needs extending. `MatchEnded` is built in TWO places: pure `MatchProjector`
  (natural end) and pure `MatchCommands` (resign/draw-accept/timeout), both from LiveMatchState.
- `user_events.proto` already has a `MatchResultRecorded` payload ("emitted by Match Manager") —
  predates the CDC decision; using it would make match-manager a second producer on
  user.events.v1, violating the CDC single-writer rule. NOT used; leave removal to 09 cleanup.

## Decisions
1. **Design: user-service consumes `MatchEnded` from `match.events.v1`** (task's recommended
   path). Stateless consumer — requires enriching `MatchEnded` with participants.
2. **Contract change** (match_events.proto, backward-compatible additions):
   - `MatchCreated`: `optional double white_bot_elo = 9`, `optional double black_bot_elo = 10`
     (engine elo snapshot at creation; unset for humans/external).
   - `MatchEnded`: `Player white = 4`, `Player black = 5`, `MatchSource source = 6`,
     `optional double white_bot_elo = 7`, `optional double black_bot_elo = 8`.
   - `Player` stays pure identity. No user_events changes.
3. **Bot elo resolved once at creation** in `MatchService.CreateMatchAsync` via existing
   `Bots.BotsClient.ListBots` (single impure site), then flows MatchCreated → LiveMatchState
   (new fields Source/WhiteBotElo/BlackBotElo) → MatchEnded, all pure. Replays of old
   MatchCreated (no elo fields) → null → user-service falls back to rating 0 (matches old
   unknown-bot behavior).
4. **Idempotency: capped `rated_matches` list stored ON the users row** (new TEXT column, JSON
   array of recent match ids, cap ~64, newest first; migration in database-service). The rating
   update + dedupe marker land in ONE row UPDATE → atomic by construction (no dedupe
   table = no two-write crash window; Postgres Insert self-assigns ids so insert-dedupe isn't
   available anyway). Redelivery → matchId found → skip. Kafka offsets are the primary
   guarantee; the set guards the redeliver/crash window.
5. **Opponent rating snapshots**: user-service reads BOTH human rows from its own (authoritative)
   store before applying either update — no replica, no GetUser RPC. Bot opponent = event-carried
   bot elo with RD=50 constant (moves to user-service).
6. user-service consumer: `Kafka/MatchEndedConsumer.cs` BackgroundService (excluded glue,
   Protobuf serde, group `user-service-rating`, gated by config flag like the CDC relay) →
   maps proto → internal `MatchEndedFact` DTO → fully-tested pure-ish handler (new
   `Rating/`-adjacent class) that does rules: external→skip, bot-vs-bot→nothing (no human side),
   per-human outcome from status+color, dedupe, snapshot-then-update.

## Sequencing (publish handoff splits the work)
- **Session A (this one):** api-contracts proto change (buf lint/breaking) → STOP for user to
  tag/publish vX.Y.Z; meanwhile: database-service migration (no proto dep), user-service pure
  handler + idempotency + tests against internal DTO (no new proto dep), docs, CONTRACT_NOTES.
- **Session B (after publish):** bump Maichess.PlatformProtos in user-service + match-manager
  (+ any other consumers per conventions), write user-service consumer glue, match-manager
  enrichment (LiveMatchState/MatchProjection/MatchCommands/MatchProjector/CreateMatchAsync +
  tests), deploy config (Helm env for the new consumer flag) in maichess-deploy, verify, stryker.

## Blockers
- Contract publish handoff: after editing match_events.proto the user must tag/push v* on
  maichess-api-contracts and the new Maichess.PlatformProtos version must exist before consumer
  code can compile.
