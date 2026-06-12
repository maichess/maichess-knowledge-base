# External Games

**Status:** Built for **two providers** in `maichess-tournament-bridge-service`:
**tournament-server** (NDJSON tournaments + games) and **Lichess** (Bot API). Both run the same
engine-drives/we-mirror model below, behind the `IExternalProvider` protocol seam
([16-lichess-bridge-compatibility](../../tasks/planned/16-lichess-bridge-compatibility.md)).

## Decision

External games use the **engine-drives, we-mirror** model. A **tournament bridge
service** opens the external provider's game stream, calls the maichess Engine
for each of our turns, submits the move to the provider, and writes a read-only
`external` match to match-db so the game surfaces in Watch and Past Matches.

The bridge is the only service that talks to external providers. No other
maichess service knows about external APIs — the bridge translates between the
provider's protocol and the maichess internal contracts (gRPC to Match Manager
and Engine).

## Architecture

```
              ┌──────────────┐
              │    Client    │
              └──────┬───────┘
                     │ REST + SSE
              ┌──────▼───────┐
              │   Tournament │
              │    Bridge    │
              └──┬───┬───┬───┘
       HTTP/     │   │   │    gRPC
      NDJSON     │   │   │
  ┌──────────┐   │   │   │  ┌──────────────┐
  │Tournament │◄──┘   │   └─►│Match Manager │
  │  Server   │       │      └──────────────┘
  └──────────┘        │
               ┌──────▼──────┐
               │   Engine    │
               └─────────────┘
```

### Pure vs IO boundaries

The core game-driving logic (`GameDriver`) is a pure function:

- **Input:** game event (position, turn, clock) + our color + bot config
- **Output:** command (SubmitMove, SyncMatch, FinalizeMatch)

All IO (HTTP to tournament server, gRPC to Engine/Match Manager) lives in a
separate `GameDriverIO` adapter that executes commands. This keeps the
turn-detection, state-mapping, and time-budget logic testable without mocking
network calls.

Similarly, `TournamentOrchestrator` consumes the tournament NDJSON stream and
delegates per-game lifecycle to `GameDriver` instances — its stream-wiring is IO,
but the "which game am I in, what round is it" logic is pure.

## Provider seam (`IExternalProvider`)

Adding a provider means implementing one interface — the engine-drives loop is written
against it, not against any wire protocol:

```
IExternalProvider:
  Name                       // mirrored into Match.external_provider
  StreamGameAsync(ref, ct)   // → IAsyncEnumerable<GameUpdate>  (parse-event→domain)
  SubmitMoveAsync(ref, uci)  // submit our move
```

`GameUpdate` is the provider-normalized snapshot the loop consumes: move list, current
FEN, side to move, status, **millisecond** clocks, our colour and the opponent name. Each
provider owns the translation from its own wire format into `GameUpdate`:

- **tournament-server** — NDJSON `gameState`/`move` events already carry a FEN; clocks are
  **seconds** and are converted to ms. (Pre-existing `TournamentServerClient` /
  `TournamentOrchestrator` path.)
- **Lichess** (`LichessProvider`, Bot API) — the stream is `gameFull` then `gameState`
  NDJSON and **never carries a FEN**, only the running UCI move list. The position is
  reconstructed by replaying the moves from `initialFen` with the pure `ChessPosition`
  (castling rights / en passant / move counters included). Clocks are **already
  milliseconds** and pass through verbatim — no `*1000`. Parsing lives in the pure
  `LichessEventParser` (100% covered); the typed `HttpClient` (`LichessProvider`) is the
  only IO and is excluded from coverage and tested via a faked `IExternalProvider`.

The `LichessGameBridge` runs one game end-to-end: it opens the stream, creates the mirror
match from the first `gameFull` (so registration can return a watchable `match_id`
immediately), asks the Engine for a move on our turn, submits it, and mirrors every state.
`GameDriver` (action decision, time budget, winner→status) is reused untouched; only the
adapter differs. Mirroring goes through the `IExternalMatchMirror` seam, which exposes
**create + sync only and no result-recording**, so external games are unrated by
construction.

Registration, two entry points on `LichessRegistrationService` (both validate bot + token,
launch the bridge, return the maichess `match_id`):

- **Attach** — `POST /external/lichess` (`bot_id`, `lichess_token`, `game_id`): drive a game
  that already exists on Lichess.
- **Challenge** — `POST /external/lichess/challenge` (`bot_id`, `lichess_token`, `opponent`,
  + clock/level/rated): *create* the game first via `ILichessChallenger`
  (`POST /api/challenge/{user}` or `/api/challenge/ai` for Stockfish), then drive the returned
  game id. `opponent: "ai"` starts immediately; a user challenge starts once accepted (the
  provider's initial game-stream connect retries on 404 to absorb the acceptance gap). Clock
  inputs here are **seconds** (Lichess's challenge-API unit), unlike the ms game-stream clocks.

## Provider auth

The bridge mints JWTs for each provider instance. For the maichess-hosted
tournament server, it shares the `TOURNAMENT_JWT_SECRET`. For external servers
the user supplies either the JWT secret (so the bridge can mint) or pre-minted
tokens.

Two identities per tournament registration:
- **Director** (`isBot: false`) — creates and manages the tournament
- **Bot** (`isBot: true`) — joins, streams games, submits moves

## External matches in match-db

External games are created via `CreateMatch(source = EXTERNAL, external_provider
= "tournament-server" | "lichess")` and updated via `SyncExternalMatch`. This path:

- **Skips** move validation (the provider already validated)
- **Skips** bot-move scheduling (the bridge drives moves externally)
- **Does** broadcast socket events (`MoveMadeEvent`, `MatchEndedEvent`) so Watch
  spectators get real-time updates through the existing pipeline
- **Does not** call `RecordMatchResult` — external games are **unrated** and do
  not affect any player's W/L/D or Glicko-2 rating

Each mirrored match carries an `external_ref` linking back to the provider's
game ID, enabling the client to cross-reference tournament context.

## Provider contracts

`tournament-server/` is treated as an external, read-only contract. The only
permitted modification is adding auth registration endpoints (`POST
/api/auth/register`) so the bridge can obtain JWTs programmatically. All other
tournament-server behavior is consumed as-is via its OpenAPI spec.

The **Lichess Bot API** (https://lichess.org/api#tag/Bot) is consumed as-is: nothing is
generated. The bridge uses `GET /api/account` (resolve our colour), `GET /api/bot/game/
stream/{gameId}` (per-game NDJSON), `POST /api/bot/game/{gameId}/move/{uci}` (submit), and
`POST /api/challenge/{user}` / `POST /api/challenge/ai` (create a game).
Auth is the user-supplied bot-account OAuth token (bearer), passed per game — there are no
director/bot JWTs and no tournament concept, so Lichess registration is a single game
rather than a tournament lifecycle.

## Where it lives

- **Knowledge base:** this document
- **Contracts:** `Player.external_name` + `CreateMatchRequest.source/external_provider/external_ref`
  + `Match.external_ref` + `SyncExternalMatch` RPC in `protos/match-manager-service/v1/matches.proto`;
  bridge REST spec in `rest/tournament-bridge.md` (tournament endpoints + `POST /external/lichess`).
  Lichess added no proto change — `external_provider="lichess"` reuses the existing fields.
- **Services:** `maichess-tournament-bridge-service` (`IExternalProvider` seam: tournament-server +
  Lichess; `ChessPosition`, `LichessEventParser`, `LichessGameBridge`), changes in
  `maichess-match-manager-service`
- **Client:** `/tournaments` routes, external match badges in Watch/Past Matches, and a **Play
  on Lichess** Dev tool (`/tools/lichess` → `LichessPlayForm` + `useLichessPlay`) that challenges
  an opponent (AI/user) or attaches to an existing game and links straight to Watch. External
  matches render through the existing read-only viewer + badge.
