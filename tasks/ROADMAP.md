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
| 07 | Dev "All games" browser | ⬜ | [spec](planned/07-dev-all-games-browser.md) | — |
| 08 | Fix: Past Matches list | ✅ | [spec](implemented/08-fix-past-matches.md) | [match-history-and-stats](../knowledge/domain/match-history-and-stats.md) |
| 09 | Caching 1 — immutable finished-match read model | ✅ | [spec](implemented/09-caching-immutable-match-read-model.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 10 | Caching 2 — Debezium CDC for `user.events` | ✅ | [spec](implemented/10-cdc-debezium-user-events.md) | [change-data-capture](../knowledge/architecture/change-data-capture.md) |
| 11 | Caching 3 — user read models (Redis + KTable) | ✅ | [spec](implemented/11-user-read-models-redis-and-ktable.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 12 | Caching 4 — analysis L1 + leaderboards (ZSET) | ✅ | [spec](implemented/12-redis-l1-and-leaderboards.md) | [caching-and-read-models](../knowledge/architecture/caching-and-read-models.md) |
| 13 | Caching 5 — search-service (Elasticsearch) | ✅ | [spec](implemented/13-search-service-elasticsearch.md) | [search-service](../knowledge/services/search-service.md) |
| 14 | Anti-cheat service | ✅ | [spec](implemented/14-anticheat-service.md) | [anticheat-service](../knowledge/services/anticheat-service.md) |
| 16 | Lichess compatibility for tournament bridge | ⬜ | [spec](planned/16-lichess-bridge-compatibility.md) | [external-games](../knowledge/domain/external-games.md) |

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
