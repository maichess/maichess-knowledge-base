# Anti-Cheat Service

**Status:** Accepted — implementation in `tasks/14` (two iterations)
**Relates to:** [event-driven-architecture.md](../architecture/event-driven-architecture.md),
[caching-and-read-models.md](../architecture/caching-and-read-models.md),
[match-history-and-stats.md](../domain/match-history-and-stats.md)

## Context

The platform has finished-game data and a chess Engine that returns a best move + centipawn
evaluation and streams `AnalyzePosition`. That is enough to analyse how closely a player's moves
track engine choices and flag likely engine assistance. Matchmaking should let players opt out of
being paired with previously-flagged cheaters, and devs need a way to clear false positives.

## Decision

A **new `maichess-anticheat-service`** consumes match facts from Kafka, scores games with a
**combined detector** (engine correlation primary + statistical features), flags players over
threshold, and propagates the flag via events so matchmaking and the user read models see it.

### Detection (combined)

- **Engine correlation (primary):** replay each finished game through the Engine; per ply compute
  centipawn loss vs the engine line and whether the played move matched a top engine move.
  Aggregate over enough games into a suspicion score.
- **Statistical features:** move-time variance, accuracy spikes, rating-vs-performance outliers,
  folded into the same score.
- **Pre-moves matter.** The client supports pre-moves (a move committed before the opponent has
  moved), which produce near-zero think time that is *legitimate*. The detector must read the
  pre-move signal from the move events and **exclude pre-moved plies from timing analysis** (and
  weight them appropriately in correlation) so fast pre-moves are not scored as suspicious.

### Two iterations

1. **Post-game analysis (iteration 1).** Consume `MatchEnded`; pull the game (by id) and analyse
   asynchronously. This is the batch/correctness baseline.
2. **Live in-game flagging (iteration 2).** A **stream processor** over the move events
   (`match.events` / `MoveApplied`) scores incrementally *during* play — the more interesting
   Kafka use case. It maintains per-(match, player) running state and can raise an early
   suspicion signal mid-game. Same pre-move handling applies; live scoring is advisory and is
   reconciled by the authoritative post-game pass.

### Storage — dedicated db, no duplication of match-db

- A dedicated **`anticheat-db`** (a DatabaseService instance) stores **cases, scores, evidence
  pointers, and audit/unflag history** only.
- It **references** match-db by `match_id` / ply index rather than copying moves, FENs, or game
  documents. The analyser reads game data from match-db on demand; `anticheat-db` holds the
  *verdict and its provenance*, not a second copy of the game. (If a no-duplication design proves
  impossible, fall back to an `anticheat_cases` collection in match-db — but reference, don't
  copy, by default.)

### Flag propagation

- A flag change emits to **`cheat.events.v1`** (keyed by userId). The user read models
  (Redis replica + the Match Maker KTable) consume it and set a `flagged` field — so matchmaking
  filters flagged players with **no extra RPC**, consistent with
  [caching-and-read-models.md](../architecture/caching-and-read-models.md).
- The boolean flag is the only thing that propagates; the evidence stays in `anticheat-db`.

### Matchmaking toggle

- The match-finding page gets a per-search toggle: **allow / disallow being matched with
  previously-flagged players** (default: disallow). Match Maker reads each candidate's `flagged`
  status from its KTable/replica and excludes flagged players when the searcher disallows them
  (and excludes a flagged *searcher* from non-flagged opponents who disallow).
- This is a matchmaking *filter*, not a ban: flagged players can still play (e.g. against others
  who allow them, or bots).

### Dev unflag

- The Dev tab gets an overview of flagged players with their case/evidence summary and a
  **remove-flag** action. Unflagging writes an audit entry to `anticheat-db` and emits a
  clearing `cheat.events` so the read models drop the flag. Dev-gated via the existing `dev_mode`
  mechanism.

## Contracts

- New events `cheat.events.v1` schema in `maichess-api-contracts/events/`.
- New service contract: REST for the dev overview/unflag + matchmaking toggle wiring; proto for
  any internal RPC. Specified in api-contracts before implementation, per project policy.

## Deployment

- New Helm components: `maichess-anticheat-service` Deployment + an `anticheat-db` DatabaseService
  instance (Mongo) + its topic(s). Engine must be reachable (it drives correlation; expect added
  Engine load — analysis is async and rate-limited).

## Dependencies

| Direction | Service | Purpose |
|---|---|---|
| Inbound | Kafka `match.events` / `MatchEnded` | trigger analysis (both iterations) |
| Outbound | Engine | `AnalyzePosition` / best-move for correlation |
| Outbound | match-db (DatabaseService) | read game data by id (no copy) |
| Outbound | anticheat-db (DatabaseService) | cases/evidence/audit |
| Outbound | Kafka `cheat.events.v1` | flag propagation to read models |
| Outbound | User service `Users.GetUser` | dev gate (`dev_mode`) for the Dev REST surface |
| Inbound | Match Maker | reads `flagged` from its read model for the toggle |
| Inbound | Client Dev tab | overview + unflag (dev-gated) |

## Implementation decisions (task 14 — DONE)

Status: **Implemented.** `maichess-anticheat-service` (ASP.NET, flat layout like
search-service). The decisions below resolve the design above into concrete choices.

### Detection internals

- **Scoring is a layered pure core** (`Detection/`, 100% tested): `EngineCorrelationDetector`
  (weighted top-1/2/3 engine-match rate over non-book plies, normalised against an
  honest-play baseline so only the excess scores), `StatisticalDetector` (think-time
  coefficient of variation — low CV = machine-steady), `CombinedScorer` (per-game
  `0.7·correlation + 0.3·statistical`; per-case = mean over a sliding window of the last
  *K* games + an **accuracy-spike** bump; flags at threshold with ≥ MinGames), and the
  iteration-2 `LiveScorer` (Welford incremental timing, one advisory signal per
  (match, player)). All thresholds live in `DetectionOptions` (config-bound).
- **Pre-move handling** is the load-bearing anti-false-positive rule: pre-moved plies are
  dropped from *all* timing features (statistical + live) and down-weighted in
  correlation. Because the `premove` bit is client-asserted it can only ever *reduce
  timing* suspicion — it never touches engine-correlation evidence, so asserting it on
  every move cannot mask engine assistance. Tested explicitly (heavy pre-mover is never
  flagged on timing; never raises a live signal).
- **Timing comes from the event stream, not the match doc** (match docs store no per-move
  timestamps): a pure `Stream/MatchStreamProjection` folds `match.events.v1` into
  per-match `MatchTimingState` (think times = `MoveApplied.applied_at_ms` deltas), shared
  by both iterations. If that state is cold when a match ends, the post-game pass degrades
  to **correlation-only** rather than failing.

### Engine load

- Post-game analysis is a single bounded `Channel` drained **sequentially** by one worker,
  and the engine analyzer sleeps a configured delay between `AnalyzePosition` calls; book
  plies are never sent to the engine. The queue drops (logged) under flood — analysis is
  rebuildable by replaying `match.events.v1` with a fresh consumer group.

### Statistical feature substitution

- The ADR lists "rating-vs-performance outliers". Ratings aren't available to this service
  without consuming another read model, so the implemented statistical signal is the
  **accuracy-spike** feature (a game whose correlation jumps far above the player's own
  prior average). The seam (`DetectionOptions` + per-game verdicts) supports adding a
  rating replica later. Recorded in the service `CONTRACT_NOTES.md`.

### Storage & flag propagation

- `anticheat-db` is a DatabaseService **instance** (Mongo) with collections `cases`
  (one per user: status open→flagged→cleared, score, per-game verdicts with `match_id` +
  `suspicious_plies`) and `audit` (every flag/unflag/live-suspicion with actor). It
  **references** match-db by id only — no moves/FENs/game docs are copied. The
  `anticheat` migration set lives in `maichess-database-service`.
- `cheat.events.v1` is **born Protobuf** (the Avro stub was retired; the topic was never
  live). Only `PlayerFlagged`/`PlayerUnflagged` set the durable `flagged` bit on the read
  models; `LiveSuspicionRaised` is advisory and consumers ignore it for the flag. The
  match-manager Redis user replica (`CheatFlagProjection`/`CheatFlagConsumer`) and the
  Match Maker in-memory `CheatFlagStore` (replacing the never-populated KTable `Flagged`
  field) both consume it.

### Matchmaking toggle

- Per-search `allow_flagged` on `POST /queue` (default false). Admissibility for a pair
  is symmetric: each flagged side needs the other's consent (`MatchingService.IsAdmissible`),
  applied in **both** the skill and FIFO paths; the FIFO fallback re-reads the waiting
  list so a lost skill-pair race still pairs whoever remains. A filter, not a ban.

### Live signal (iteration 2)

- Live scoring is **timing-only** (no live engine calls — that would multiply Engine load
  per ongoing game). A raised signal surfaces as a `LiveSuspicionRaised` event + a
  `live_signals` counter and `audit` row visible in the Dev overview. The authoritative
  flag always comes from the reconciled post-game pass.

### Premove contract follow-up

- The `premove` fields on `MoveSubmitted`/`MoveApplied` and the REST `premove` flag were
  added to api-contracts but require the **vNext (0.7.0)** publish. Until consumed, the
  stream fold takes the premove bit as a parameter (defaulting false); the one-line flip +
  match-manager's premove production are documented in the service `CONTRACT_NOTES.md`.
