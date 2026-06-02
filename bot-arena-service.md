# Bot Arena Service

## Role

The bot arena service lets users spawn bot-vs-bot **setups** and collects their
results. It owns setup expansion, color alternation, tournament bracket
progression and tie-breaking, the global concurrency limit, and result storage.
It does **not** create matches itself — it spawns every game through Match
Maker's existing bot-vs-bot path so matchmaking stays the single owner of match
creation, and it observes outcomes through Match Manager. Live games are watchable
through the ordinary Watch flow; finished setups are stored as collections whose
typed result views the client renders.

## Why orchestration is a separate service

Match Maker's job is to turn a request for a game into one match. Driving a whole
tournament — pairing winners across rounds, aggregating best-of stages, capping
concurrency, persisting partial state — is a different, stateful concern. Keeping
it out of Match Maker preserves that service's single responsibility and lets the
arena own its own database without reaching into match-db.

## Technology

ASP.NET (net10.0), consistent with Match Maker and Match Manager. REST-only
surface (the client talks REST; nothing calls the arena over gRPC in this
release), plus gRPC/HTTP **clients** to its dependencies. `arena.proto` is the
canonical data-model contract.

## Persistence

A dedicated **`arena-db`** database-service instance (per the knowledge-base
database-service principle), reached over the generic CRUD gRPC contract with
`Struct` records — never a direct DB driver. Three collections: `collections`
(setup + status + resolved config + tournament seeding), `games` (one record per
spawned game with its match id, colors, start FEN, outcome, final clocks/FEN, and
round/pairing tags), and `settings` (the global concurrency limit as a single
keyed record, since the generic Insert assigns its own id).

## Setup-type semantics

FEN-list semantics everywhere: an empty list or `["standard"]` means the standard
start position; the standard position is labelled "Standard", others "FEN N".

- **Single** — for each FEN, `games_per_fen` games. With `keep_switching_colors`
  the white/black assignment flips every game, continuously across FENs;
  otherwise it is fixed.
- **Matrix** — every unordered bot pair, for each FEN, `games_per_fen` games,
  with colors alternating per game continuously within a pairing.
- **Tournament** — all bots seeded into a **random** single-elimination bracket
  (Fisher-Yates with an injectable RNG). A non-power-of-2 field is reduced to a
  power of two in the first round by giving the leading seeds **byes**. Each stage
  plays the creator-chosen `fens_per_stage` positions (the first N of the pool);
  `color_mode` is either `both_colors` (each position played twice, colors
  swapped) or `random` (each position played once, colors chosen by RNG).

### Tournament stage winner — tie-break ladder

A stage winner is decided in this order; each step only applies when the previous
did not separate the bots:

1. **More wins.**
2. **Fewer rounds to win** — the bot whose last win came earlier (lower 1-based
   game index); a winless bot ranks last here.
3. **Greater aggregate clock advantage** — summed remaining-clock difference
   across the stage's games.
4. **Greater aggregate final material** — summed material difference (P1 N3 B3 R5
   Q9) at each game's final position.
5. **A seeded coin flip** — guarantees a decisive winner so the bracket always
   advances.

Stalemate (and fifty-move, threefold, insufficient material) is a **draw**,
mirroring Match Manager's `MatchStatus`, which has no separate stalemate state.

### Derived bracket

The bracket is **not** stored. Only the random seeding is persisted; every
stage's winner is recomputed from its games with a deterministic per-stage RNG
(seeded from the collection id + round + pairing), so the bracket — and the
champion — are a pure function of the seeding and the games played. This keeps
persistence to flat collection/game records and makes the progression logic
purely testable.

## Concurrency model

A single **global** concurrency limit caps how many arena games run at once
across all setups and all users. It is editable by any authenticated user (not
per-user) and defaults to 4 until set. A background poller observes running
games; as they finish, queued games are launched up to the spare capacity
(`LaunchableCount = clamp(limit - running, 0, pending)`).

## Match attribution and auth

Each spawned match is attributed to the user who created the setup. The arena
mints a short-lived JWT signed with the shared `Jwt:Key` (subject = the creator's
user id) and calls Match Maker's bearer-protected `POST /matches/bot-vs-bot`,
which now forwards that identity as `CreateMatchRequest.created_by`. Custom start
positions ride the same path via the new `start_fen` field.

## Client (Dev section)

The Dev section is `dev_mode`-gated (see `dev-mode.md`). The client-side arena UI
lives under `app/dev/` with server-side `requireDevUser()` guards and follows the
standard proxy pattern (`app/api/dev/arena/` routes forwarding to the arena service
with Bearer auth via `getBearerToken()`).

- **Dev landing** (`/dev`): links to spawn setups and results, plus a global
  concurrency limit control editable by any dev user.
- **Spawn** (`/dev/arena/new`): tabbed form for tournament, matrix, and single
  setups. Multi-select bots (from `useBots`), editable FEN list, time-format
  picker (from `useTimeFormats`). Submits create the collection via proxy and
  redirect to the collection detail.
- **Results** (`/dev/arena` list + `/dev/arena/[id]` detail): running collections
  poll for live progress. The detail page renders **type-specific views**:
  - **Tournament → bracket tree**: rounds as columns, pairings as nodes with
    scores and per-game links.
  - **Matrix → grid**: bots on both axes, cells showing aggregate scores, with
    a full game list.
  - **Single → labelled list**: score summary plus per-game rows with FEN label,
    bot colors, and result.
- **Live games** link into the existing Watch viewer (`/watch/[id]`) — the arena
  does not have its own board.

Models in `lib/models/arena.ts`, hooks in `lib/hooks/useArena*.ts`.

## Testing

Pure domain (expansion, scoring, tie-break ladder, bracket, launch planning) and
the orchestration state machine are 100% covered with Reqnroll BDD; low-level
calculations and adapters use xUnit. The database-service repository, Match
Maker/Manager/Engine client adapters, the fire-and-forget poller loop, REST
endpoint handlers, and DTOs are `[ExcludeFromCodeCoverage]` per the platform
rules — the scheduling *decisions* they act on stay pure and tested.
