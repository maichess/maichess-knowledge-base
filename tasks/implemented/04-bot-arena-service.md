# 04 — Bot Arena service (tournament / matrix / single setups)

**Goal:** A new ASP.NET microservice, `maichess-bot-arena-service`, that lets users
spawn bot-vs-bot **setups** and collects their results. Three setup types:

- **Tournament** — all bots seeded into a **random pairing** bracket; each stage is a
  **best-of-3** where the games use different start positions from a **FEN list**, and
  **for each FEN each bot plays once as white and once as black**. Winner advances.
- **Matrix** — round-robin: **each bot vs each other bot**, over a FEN start-position
  list, **N games per FEN**, **always alternating colors** for each pairing/setup.
- **Single** — one match-up between two bots: **FEN list** (default = standard start),
  **number of games per FEN**, and a **keep-switching-colors** toggle.

Live games are watchable through the **existing Watch** flow (they are ordinary
bot-vs-bot matches in match-manager). Finished setups are stored as **collections**
whose results prompt `05` renders.

> Read `conventions.md` first. Depends on nothing functionally but is
> gated in the UI by `01`; the results UI is `05`.

## Background to read first

- match-maker bot-vs-bot entry point: `POST /matches/bot-vs-bot` with body
  `{ whiteBotId, blackBotId, timeFormatId }` → `{ matchId }`
  (`services/maichess-match-maker-service/Rest/QueueEndpoints.cs`,
  `Rest/BotMatchRequest.cs`/`BotMatchResponse.cs`,
  `Queue/QueueingService.CreateBotVsBotMatchAsync`). The arena spawns games through
  this. **It does not call match-manager `CreateMatch` directly** — go through
  match-maker so matchmaking stays the single owner of match creation.
  > Custom start positions are **required** for the FEN-list setups, so this prompt
  > adds first-class `start_fen` support to the bot-vs-bot path — see
  > "Start-FEN support" under Contract changes below.
- Live state / completion: match-manager `GetMatch` / `ListMatches` / `MatchEvent`
  SSE (`protos/match-manager-service/v1/matches.proto`). The arena observes match
  outcomes to advance brackets / fill the matrix / aggregate best-of-3.
- engine bot list: `Bots.ListBots` (`protos/engine-service/v1/bots.proto`) for the
  selectable bots.
- **Persistence rule:** like user-service (`UsersService.cs` using
  `Database.DatabaseClient`), the arena service MUST persist through a dedicated
  **database-service** instance over the generic gRPC CRUD contract
  (`protos/database-service/v1/database.proto`) — **never a direct Mongo/SQL driver.**
  Provision a **new `arena-db` database-service instance** (its own configured domain,
  per the knowledge-base database-service principle) rather than overloading match-db.
- Existing test/coverage patterns: copy the match-manager / match-maker service
  layout (Reqnroll BDD for business logic, unit tests for adapters, NSubstitute,
  `FakeLogger<T>`, coverlet, Stryker.NET). See those services' memories/READMEs.

## Contract changes (`maichess-api-contracts`)

### Start-FEN support on bot-vs-bot (match-maker + match-manager)

The FEN-list setups require games to begin from arbitrary positions, so add
first-class custom-start-position support to the existing bot-vs-bot path:
- match-maker REST `POST /matches/bot-vs-bot` (`rest/match-maker.md`,
  `Rest/BotMatchRequest.cs`): add an optional `start_fen` field.
- match-manager `CreateMatchRequest`
  (`protos/match-manager-service/v1/matches.proto`): add an optional
  `string start_fen = 4;`.
- **Default behavior:** when `start_fen` is omitted, empty, or the literal
  `"standard"`, the match begins from the standard initial position exactly as today
  (this is the only behavior existing callers ever trigger, so they are unaffected).
  When a FEN is provided, the match starts from it — `current_fen` and `FenHistory[0]`
  are seeded from that FEN. Validate it (delegate to move-validator if a legality
  check is wanted) and reject invalid positions with `400`.
- This is a contract change to two existing services — follow the versioning handoff
  in `00` (publish a new tag, bump `Maichess.PlatformProtos` everywhere).

### Arena service contract

Create `protos/bot-arena-service/v1/arena.proto` (package `maichess.bot_arena.v1`,
`csharp_namespace = "Maichess.BotArena.V1"`) and `rest/bot-arena.md`.

Model (names indicative — keep proto style rules):
- `enum SetupType { SETUP_TYPE_UNSPECIFIED; TOURNAMENT; MATRIX; SINGLE; }`
- `GameCollection` — `id`, `name`, `SetupType type`, `created_by`, `status`
  (`pending|running|finished`), `created_at_ms`, `finished_at_ms`, the resolved
  config, and per-type result payload (bracket / matrix / list).
- Config messages:
  - `TournamentConfig { repeated string bot_ids; repeated string fen_list; TimeFormat time_format; }`
    (best-of-3 + per-FEN both-colors are fixed semantics, documented).
  - `MatrixConfig { repeated string bot_ids; repeated string fen_list; int32 games_per_fen; TimeFormat time_format; }`
    (alternating colors fixed).
  - `SingleConfig { string white_bot_id; string black_bot_id; repeated string fen_list; int32 games_per_fen; bool keep_switching_colors; TimeFormat time_format; }`
  - FEN list semantics: empty / `["standard"]` ⇒ standard start position.
- `GameResult` — `match_id`, `fen`/`fen_label`, `white_bot_id`, `black_bot_id`,
  `MatchStatus result`, ordering metadata so `05` can render trees/matrix/list.
- Result views: `TournamentBracket` (rounds → pairings → best-of-3 game results +
  winner), `MatrixTable` (bot×bot cells with aggregate score), `SingleSeries` (ordered
  list of labelled games).
- RPCs / REST:
  - Create a setup (one per type, or a `CreateCollection` with a `oneof` config).
  - List collections (filter by status; finished ones for the Results tab).
  - Get a collection (with its typed result view + live progress).
  - **Global concurrency limit** get/set — a single global integer, editable by any
    user (not per-user). Store it as a singleton config row via the database-service.

Versioning handoff per `00`: publish a new tagged contracts version; bump
`Maichess.PlatformProtos` everywhere (arena service is a new consumer; match-maker +
match-manager change too if `start_fen` is added).

## Service implementation (`services/maichess-bot-arena-service/`)

Scaffold to match the other ASP.NET services (sln + main project + `.Tests`,
`nuget.config`, `README.md`, `CONTRACT_NOTES.md`, `.config/dotnet-tools.json` with
Stryker, `stryker-config.json` in the test project).

Layered design (keep side effects at the edges; domain logic pure & fully tested):
- **Domain (pure, 100%-tested):**
  - FEN-list normalization (empty/standard handling, labels for display).
  - **Setup expansion** → the exact ordered list of `(whiteBotId, blackBotId, fen)`
    games for each type:
    - Single: for each FEN, N games; if `keep_switching_colors`, alternate colors per
      game (and across FENs); else fixed colors.
    - Matrix: each unordered bot pair, for each FEN, N games, alternating colors.
    - Tournament stage: best-of-3 = for each of the (up to 3) FENs, both color
      assignments — encode the "each FEN once as white and once as black" rule and the
      best-of-3 aggregation (first to 2 points; draws = ½; tie-break rule documented).
  - **Bracket progression** (random seeding with an injectable RNG/seed for
    deterministic tests), **matrix aggregation**, **single-series labelling**.
- **Orchestration layer:** drives game creation via the match-maker bot-vs-bot client
  (passing each expanded game's `fen` as `start_fen`, and omitting it for the standard
  position), **respecting the global concurrency cap** (no more than N arena games in flight at
  once — a scheduler that launches queued games as running ones finish). Observe
  completion via match-manager (poll `GetMatch`, or subscribe to match end). Advance
  state, persist progress, and finalize the collection.
- **Persistence:** all reads/writes go through the `arena-db` **database-service**
  client (generic CRUD with `Struct` records, exactly like `UsersService.cs`).
  Collections, per-game results, and the global concurrency setting are stored there.
- **REST adapter + gRPC** (if any inter-service RPC is needed): thin, `[ExcludeFromCodeCoverage]`.

Coverage exclusions (mirror the platform rules): REST endpoint adapters, the
database-service-backed repository wrapper (live dependency), any fire-and-forget
scheduler loop body that can't be tested deterministically (keep the *scheduling
decision* logic pure and tested — only the loop wrapper is excluded), `[LoggerMessage]`
partials, DTO records, `Program.cs`/`*.g.cs`. Everything else **100%**. Wire Stryker
to mirror these exclusions.

## Deployment config (`maichess-deploy/helm/maichess/`)

- Add a service `Dockerfile` in `services/maichess-bot-arena-service/`.
- `templates/bot-arena-service.yaml` using `_app-deployment.tpl` + `_app-service.tpl`
  (copy a peer like `match-maker-service.yaml`).
- A `templates/arena-db.yaml` database-service instance (copy `match-db.yaml`/
  `user-db.yaml` pattern) plus its backing store, configured for the arena domain.
- Add image/resources/env entries in `values.yaml`, `values-staging.yaml`,
  `values-prod.yaml`; wire `hpa.yaml`, `pdb.yaml`, and `ingress.yaml` to match peers
  (the arena REST API must be reachable by the client like other services).
- Add the build/publish step to `.github/workflows/deploy.yml`.

## Verification

- `cd services/maichess-bot-arena-service && dotnet test -p:CollectCoverage=true` →
  100% on non-excluded; `dotnet stryker` → review surviving mutants in expansion /
  bracket / aggregation logic.
- Domain unit tests assert exact expansion for representative configs (e.g. single
  best-of-3 color alternation; 4-bot matrix over 2 FENs; 4-bot tournament bracket with
  a fixed RNG seed).
- Manual (after `05`): create each setup type → games appear in Watch, the global
  concurrency cap is respected, and a finished collection stores a correct typed
  result payload.

## Knowledge base

Add an ADR `maichess-knowledge-base/bot-arena-service.md`: the new service's
responsibility, why orchestration is separate from match-maker, the exact setup-type
semantics (best-of-3 both-colors, matrix alternation, single toggle), the global
(not per-user) concurrency model, and that it persists via a dedicated `arena-db`
database-service instance (no direct DB driver).
