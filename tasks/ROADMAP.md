# maichess Roadmap & Task Status

Single source of truth for **what work exists and where it stands**. Each task has a self-contained
spec under [`implemented/`](implemented/) or [`planned/`](planned/); the durable design behind it
lives under [`../knowledge/`](../knowledge/). Shared rules every spec assumes are in
[conventions.md](conventions.md).

**Legend:** ✅ shipped · 🟡 in progress · ⬜ planned · 🟥 aborted/superseded

## Feature & migration tasks

| # | Task | Status | Spec | Design doc |
|---|------|--------|------|-----------|
| 01 | Dev-mode toggle | ✅ | [spec](implemented/01-dev-mode-toggle.md) | [dev-mode](../knowledge/domain/dev-mode.md) |
| 02 | Past matches + real W/L/D stats | ✅ | [spec](implemented/02-past-matches-and-stats.md) | [match-history-and-stats](../knowledge/domain/match-history-and-stats.md) |
| 03 | Glicko-2 rating | ✅ | [spec](implemented/03-elo-glicko2.md) | [rating-glicko2](../knowledge/domain/rating-glicko2.md) |
| 04 | Bot Arena service | ✅ | [spec](implemented/04-bot-arena-service.md) | [bot-arena-service](../knowledge/services/bot-arena-service.md) |
| 05 | Bot Arena client (Dev UI) | ✅ | [spec](implemented/05-bot-arena-client.md) | — |
| 06 | External games (Lichess + tournament-server) | 🟥 | superseded → 16 | [external-games](../knowledge/domain/external-games.md) |
| 07 | Dev "All games" browser | ✅ | [spec](implemented/07-dev-all-games-browser.md) | [match-history-and-stats](../knowledge/domain/match-history-and-stats.md) |
| 08 | Fix: Past Matches list | ✅ | [spec](implemented/08-fix-past-matches.md) | [match-history-and-stats](../knowledge/domain/match-history-and-stats.md) |
| 09 | Caching 1 — immutable finished-match read model | ✅ | [spec](implemented/09-caching-immutable-match-read-model.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 10 | Caching 2 — Debezium CDC for `user.events` | ✅ | [spec](implemented/10-cdc-debezium-user-events.md) | [change-data-capture](../knowledge/architecture/change-data-capture.md) |
| 11 | Caching 3 — user read models (Redis + KTable) | ✅ | [spec](implemented/11-user-read-models-redis-and-ktable.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 12 | Caching 4 — analysis L1 + leaderboards (ZSET) | ✅ | [spec](implemented/12-redis-l1-and-leaderboards.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 13 | Caching 5 — search-service (Elasticsearch) | ✅ | [spec](implemented/13-search-service-elasticsearch.md) | [search-service](../knowledge/services/search-service.md) |
| 14 | Anti-cheat service | ✅ | [spec](implemented/14-anticheat-service.md) | [anticheat-service](../knowledge/services/anticheat-service.md) |
| 16 | Lichess compatibility for tournament bridge | 🟡 | [spec](planned/16-lichess-bridge-compatibility.md) | [external-games](../knowledge/domain/external-games.md) |
| 17 | `ListBots` in-memory cache in match-manager | ✅ | [spec](implemented/17-listbots-in-memory-cache.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 18 | Bot arena Kafka-native completion (replace poller) | ✅ | [spec](implemented/18-bot-arena-kafka-completion.md) | [event-driven-architecture](../knowledge/architecture/event-driven-architecture.md) |
| 19 | Engine timing strategy ELO calibration | ⬜ | [spec](planned/19-timing-strategy-elo-calibration.md) | — |
| 20 | Bot arena matrix: color switching toggle | ✅ | [spec](implemented/20-bot-arena-matrix-color-toggle.md) | [bot-arena-service](../knowledge/services/bot-arena-service.md) |
| 21 | Play: choose your color (human queue + vs-bot) | ✅ | [spec](implemented/21-play-color-selection.md) | — |
| 22 | Analysis read-only mode + stop surfacing gRPC "Cancelled" | ✅ | [spec](implemented/22-analysis-readonly-viewer.md) | [analysis-service](../knowledge/services/analysis-service.md) |
| 23 | Past matches: include user-initiated in-progress games | ✅ | [spec](implemented/23-past-matches-include-initiated.md) | [analysis-service](../knowledge/services/analysis-service.md) |
| 24 | Search: partial matching, searchable names, bot games | ✅ | [spec](implemented/24-search-relevance-and-partial-matching.md) | [search-service](../knowledge/services/search-service.md) |
| 25 | Fix: "All games" browser fails to load | ✅ | [spec](implemented/25-fix-all-games-fails-to-load.md) | [match-history-and-stats](../knowledge/domain/match-history-and-stats.md) |
| 26 | Knowledge "classical" bot as default analysis engine + cache | ✅ | [spec](implemented/26-classical-bot-default-analysis.md) | [analysis-service](../knowledge/services/analysis-service.md) |
| 27 | Bot arena: global capacity scheduler (pending starts + concurrency) | ✅ | [spec](implemented/27-bot-arena-global-capacity-scheduler.md) | [bot-arena-service](../knowledge/services/bot-arena-service.md) |
| 28 | Auth: premature logout / couple session to activity | ✅ | [spec](implemented/28-fix-premature-logout.md) | — |
| 29 | Strongest-bot search/eval hardening + variant-aware multi-PV analysis | ✅ | [spec](implemented/29-strongest-bot-search-eval-hardening.md) | [engine CLAUDE.md](../../services/maichess-engine-service/CLAUDE.md) |
| 30 | Strongest-bot endgame tablebase hardening (local source + caching + analysis WDL) | ⬜ | [spec](planned/30-strongest-bot-tablebase-hardening.md) | [engine CLAUDE.md](../../services/maichess-engine-service/CLAUDE.md) |
| 31 | Insights & Spark analytics (historical-game analysis) | 🟡 | [program](planned/insights/README.md) | [insights-and-spark](../knowledge/architecture/insights-and-spark.md) |

## UX bug-fix / feature batch (2026-06-12) — client changes shipped inline

A batch of small client-side reports was implemented directly (no spec needed); the
larger / backend items above (21–28) were filed as specs. Shipped inline:

- **Tools menu + Dev cleanup** — new **Tools** tab (available to all signed-in users;
  auth-only via `requireUser`) holding Bot arena, All games, and Search, moved out of
  `/dev` to `/tools/*`; `/dev` now holds only Anti-cheat + arena concurrency. Removed the **Profile** nav tab — the username/avatar in the top-right is
  now the profile link. (`maichess-client`: `Nav.tsx`, `app/tools/*`, `app/dev/page.tsx`,
  `routes.ts`, `proxy.ts`.)
- **Analysis copy PGN/FEN** — reused `ExportGamePanel` in the game analysis view
  (`AnalysisClient.tsx`).
- **"Show analysis" after a game ends** — new `AnalyseGameButton` under the result banner
  in Match and Watch views (reuses the import-from-match flow).
- **Dashboard "Continue playing"** — now only lists games where the user is actually a
  player, not merely the initiator (`app/dashboard/page.tsx`).
- **Arena collection live-vs-pending tag** — collection badge shows "live" only when a
  game is actually in flight (`running_games > 0`), else "pending"
  (`ArenaCollectionList`/`Detail`). Per-game tag delivered in task 27 (per-game
  `status` on `GameResult`).

## Event-driven (Kafka) migration — its own program

The platform's largest in-flight effort spans many services, so it is a **dedicated, ordered task
program** rather than a single spec:

- **[tasks/planned/kafka/](planned/kafka/README.md)** — 🟡 **in progress.** The program README holds
  the current state (socket fan-out, matchmaking facts, and `CreateMatch`-over-Kafka are live) and the
  ordered task list `01`–`09`: Protobuf event foundation → migrate live topics → validator & engine
  stream processors → Redis read model + projector → 202 command side → analysis → rating events →
  decommission gRPC + remove the registry. **Protobuf-first** — no new Avro is written.
- **Design:** [event-driven-architecture](../knowledge/architecture/event-driven-architecture.md) and
  [serialization-protobuf-migration](../knowledge/architecture/serialization-protobuf-migration.md).

## Insights & Spark analytics — its own program

A new capability spanning a new service, a Scala Spark batch module, and new infrastructure, so it is a
**dedicated, ordered task program** (task `31`) rather than a single spec:

- **[tasks/planned/insights/](planned/insights/README.md)** — ⬜ **planned.** Download & analyze massive
  historical chess corpora (built-in monthly [Lichess dumps](https://database.lichess.org/#standard_games),
  manual PGN upload, pluggable sources) with **Apache Spark on k8s** → most successful openings, common
  endgames/positions, and "tricky" (blunder ∩ think-time) positions. The program README holds the ordered
  task list `01`–`07`: contracts + `insights-db` → MinIO + Spark Operator → Spark ingestion/parser →
  Spark analysis jobs → .NET control plane → query API → client page. **Annotations-first** (Lichess
  `%eval`/`%clk`), Spark node-pinned + resource-capped on `maichess-mega`.
- **Design:** [insights-and-spark](../knowledge/architecture/insights-and-spark.md),
  [spark-and-minio](../knowledge/operations/spark-and-minio.md), and
  [insights-statistics](../knowledge/domain/insights-statistics.md).
- **Dependency order:** `01 → 02 → 03 → 04 → 05 → 06 → 07` (`01`/`02` independent of each other).

## Dependency order (for the original feature batch)

`01` → `02` → `03` → `04` → `05`; `07` runnable any time after `02`.

Second batch (caching + new services), after `02`:
`08` (independent, first) ; then `09` → `10` → `11` → `12` ; `13` after `10` ; `14` after `11`.
The Kafka migration runs on its own track as a dedicated program (see above).

- `03` depends on the `RecordMatchResult` path from `02`.
- `05` depends on the Dev gate (`01`) and the service (`04`).
- `07` depends only on `01` (Dev gate) and `02` (Match `created_by`/`source` + listing).
- `08` is an independent bug fix; the caching stages assume canonical user ids, so it goes first.
- `09`→`12` are the caching stages in leverage-to-risk order; `10` must precede `11` (Stage 3
  trusts the CDC-derived `user.events`). `13` needs the Mongo Debezium connector from alongside `10`.
- `14` rides the user read model from `11` (the `flagged` field) and reuses the `01` Dev gate.
- `16` extends the already-built tournament bridge; depends on nothing new.
- `17` is independent; requires only that `IMemoryCache` is available (it already is in match-manager).
- `18` depends on Kafka task `06` (match events on `match.events.v1` must be live).
- `19` is operational (run arena series, update `BotRegistry`); depends on `04`/`05` for the arena UI.
- `20` depends on `04`/`05`; requires a contract bump.
