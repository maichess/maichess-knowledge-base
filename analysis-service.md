# Analysis Service

## Role

The analysis service is the back-end for post-game and ad-hoc chess analysis. It has two
distinct responsibilities:

1. **Game persistence** — stores analysis games sourced from PGN imports, finished matches, or
   arbitrary FEN positions, keyed per user. Exposes a REST API consumed directly by the client.

2. **Engine analysis streaming** — manages per-user analysis sessions. When a client starts
   analysis for a session, the service calls the Engine service's `AnalyzePosition` endpoint and
   pushes each depth update to the client via `Socket.EmitEvent`. Analysis results for the default
   bot are cached in the database so repeat requests for the same position serve immediately from
   cache and continue live from the engine for deeper depths.

The client never connects to a separate WebSocket for analysis. All real-time analysis delivery
goes through the shared socket.io connection to the Socket Service.

## Technology

ASP.NET, consistent with Match Manager and Match Maker.

## Databases and Collections

MongoDB, same cluster used by Match Manager. The analysis service accesses two collections:

- **`analysis_games`** — analysis game documents (owned by this service).
- **`matches`** — match documents written by Match Manager (read-only access by this service,
  used only for `POST /games/from-match/{match_id}`).
- **`analysis_results`** — per-depth engine analysis results (owned by this service, used as an
  analysis cache).

## Data Models

### AnalysisGame

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `user_id` | string | Owner; all queries are scoped to this |
| `source` | enum `pgn \| match \| fen` | Origin of the game |
| `match_id` | string? | Populated when `source == match` |
| `starting_fen` | string | FEN before any moves. Standard opening position for PGN/match imports unless PGN contains a `[FEN]` header. Arbitrary FEN for `source == fen`. |
| `moves` | string[] | Full move list in UCI notation, oldest first. Empty for `source == fen`. |
| `fens` | string[] | FEN after each move, parallel to `moves`. Empty for `source == fen`. |
| `pgn` | string | Full PGN string including headers and moves in SAN notation |
| `result` | string | PGN result token: `1-0`, `0-1`, `1/2-1/2`, or `*` |
| `white` | PlayerInfo | Name string for PGN imports; `{ user_id }` or `{ bot_id }` for match imports |
| `black` | PlayerInfo | Same as white |
| `tags` | map | All PGN header key/value pairs |
| `created_at` | timestamp | |

`fens` is derived on import/save and stored so the client can navigate positions without
recomputing FENs from the move list on every request.

Games are never deleted by the user. The library grows as games are imported.

### AnalysisResult (analysis cache)

One document per engine depth update for a given position. Stored only when `bot_id` matches
the server-configured `DefaultAnalysisBotId`.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `fen` | string | Position analyzed |
| `bot_id` | string | Engine used |
| `line_count` | integer | Number of principal variations |
| `depth` | integer | Engine search depth for this document |
| `lines` | array | `[{ rank, evaluation_cp, moves[] }]` — all lines at this depth |
| `created_at` | timestamp | |

Cache lookup: `List(collection="analysis_results", filter={fen, bot_id}, limit=100)`.
Post-filter in the application layer on `line_count >= session.line_count`. Sort by `depth`.

A limit of 100 is safe given practical engine depth ceilings (~40 moves).

### analysis_meta

A single document tracking which bot's analysis is currently stored in `analysis_results`. Used
during startup to detect bot configuration changes.

| Field | Type | Notes |
|---|---|---|
| `id` | string | Always `"config"` |
| `stored_bot_id` | string | `bot_id` of the cached analysis data |

On startup: read `stored_bot_id`. If it differs from `DefaultAnalysisBotId` in config, call
`Database.DeleteWhere(collection="analysis_results")` to scrape all cached data, then update
`stored_bot_id`. This prevents stale analysis from a previous default bot persisting in the cache.

## REST API

See [`rest/analysis.md`](../maichess-api-contracts/rest/analysis.md).

### Games endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/games` | List the authenticated user's saved games |
| `GET` | `/games/{id}` | Get a single game with full move list and PGN |
| `POST` | `/games` | Import and save a game from PGN |
| `POST` | `/games/from-match/{match_id}` | Import a finished match as an analysis game |
| `POST` | `/games/from-fen` | Create an analysis game from an arbitrary FEN |

### Configuration endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/analysis/config` | Default bot, default line count, available bots |

### Session endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/sessions` | Create session (one per user; auto-cancels previous) |
| `DELETE` | `/sessions/{id}` | Destroy session |
| `POST` | `/sessions/{id}/navigate` | Jump to game move index; clears whatif |
| `POST` | `/sessions/{id}/whatif` | Play speculative UCI move |
| `DELETE` | `/sessions/{id}/whatif` | Reset whatif branch |
| `DELETE` | `/sessions/{id}/whatif/last` | Undo last whatif move |
| `GET` | `/sessions/{id}/whatif/pgn` | Export whatif branch as PGN |
| `POST` | `/sessions/{id}/analysis` | Start/restart analysis |
| `DELETE` | `/sessions/{id}/analysis` | Stop analysis |

## Analysis Session

Sessions are in-memory only (not persisted to the database). One session per user; creating a
new session destroys the previous one.

### Session state

- `session_id` — opaque identifier returned on creation
- `game_id` — the analysis game being explored
- `bot_id`, `line_count` — engine configuration for this session
- `current_index` — position in the game's move list (0 = starting position)
- `whatif_moves` — UCI moves played speculatively since the last navigation
- `whatif_fens` — FENs after each whatif move (computed incrementally via `ValidateMove`)
- `active_cts` — `CancellationTokenSource` for the current engine stream, or null

### Current FEN

- If `whatif_moves` is empty: `game.starting_fen` (when `current_index == 0`) or `game.fens[current_index - 1]`
- If `whatif_moves` is non-empty: `whatif_fens[last]`

### Analysis lifecycle

1. **Session created** — no analysis running.
2. **`POST /sessions/{id}/analysis`** — starts engine stream.
3. **Navigation or whatif move** — cancels current stream (if any), starts new stream for the new position.
4. **`DELETE /sessions/{id}/analysis`** — cancels current stream; session stays alive.
5. **Engine finishes naturally** — stream ends; `analysis_complete` event sent; session stays alive.

### Engine stream with cache

When analysis starts for a position `(fen, bot_id, line_count)`:

1. Query `analysis_results` for all cached depths matching `(fen, bot_id)` with
   `line_count >= session.line_count`.
2. Emit all cached depth updates immediately via `Socket.EmitEvent` (in depth order, trimming
   lines to `session.line_count`). This is transparent to the client.
3. Determine `max_cached_depth` (0 if no cache).
4. Open `Engine.AnalyzePosition` stream.
5. For each engine update:
   - If `depth <= max_cached_depth`: discard (already served from cache).
   - If `depth > max_cached_depth` AND `bot_id == DefaultAnalysisBotId` AND
     `line_count == DefaultLineCount`: write to `analysis_results`.
   - Emit via `Socket.EmitEvent`.
6. On engine stream end: emit `analysis_complete`.
7. On cancellation: stop silently (no `analysis_complete`).

### Whatif position_history

The `position_history` opaque blob (used by the move validator for threefold repetition) is
not stored in `AnalysisGame` documents. Whatif moves are validated with an empty initial
`position_history`. This means threefold repetition detection covers only positions within
the whatif branch itself, not positions from the preceding game moves. This is a known
limitation acceptable for analysis use.

## SAN notation

All chess notation logic (SAN ↔ UCI conversion) is handled by the Move Validator service.
The analysis service contains no chess notation or board logic.

- **PGN import:** for each SAN move, call `Moves.ValidateMoveSan(fen, san, position_history)` →
  `{ resulting_fen, position_history, uci_move }`. No local SAN parsing.
- **Match import PGN generation:** call `Moves.ConvertSequenceToSan(fen_history[0], match.moves)` → `san_moves[]` in one round-trip.
- **Whatif PGN export:** call `Moves.ConvertSequenceToSan(whatif_base_fen, session.WhatifMoves)` → `san_moves[]` in one round-trip.
- **PGN stored in `analysis_games.pgn`** uses SAN notation throughout.

## match-db access for match import

`POST /games/from-match/{match_id}` reads match data directly from match-db via
`Database.Get(collection="matches", id=match_id)`. It does not call the Match Manager gRPC
service.

Fields used from the match document:

| Field | Used for |
|---|---|
| `status` | Must not be `"ongoing"` |
| `white_user_id`, `black_user_id` | Participant check |
| `white_bot_id`, `black_bot_id` | Player info in the imported game |
| `moves` | UCI move list |
| `fen_history` | Pre-computed FENs (`fen_history[0]` = start, `fen_history[N]` = after move N) |

The `position_history` field in the match document is ignored entirely.

## gRPC Interface

The analysis service exposes **no gRPC server endpoint**. It is a pure REST server that calls
gRPC services as a client.

The `analysis.proto` file and `Analysis` service have been removed. The previously planned
Analysis WebSocket Service has been abandoned.

## Dependencies

| Direction | Service | Protocol | Purpose |
|---|---|---|---|
| Inbound | Client | HTTP REST | Game persistence and session control |
| Outbound | Engine Service | gRPC | `AnalyzePosition` |
| Outbound | Move Validator | gRPC | `ValidateMoveSan`, `GetLegalMovesSan` (SAN handling), `ValidateMove` (whatif validation) |
| Outbound | Database Service (match-db) | gRPC | `analysis_games`, `analysis_results`, `analysis_meta` collections; `matches` collection (read-only) |
| Outbound | Socket Service | gRPC | `EmitEvent` — push analysis updates to client |

Auth is handled locally via JwtBearer middleware (same shared JWT key).

## Authorization

- All game and session operations are scoped to the authenticated user.
- `POST /games/from-match/{match_id}` additionally requires the user to have been a participant
  (white or black player) of the match.

## Known limitations

- Session state is not persisted; server restarts require clients to recreate sessions.
- Whatif threefold repetition detection covers only the whatif branch itself.
