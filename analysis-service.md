# Analysis Service

## Role

The analysis service is the back-end for post-game and ad-hoc chess analysis. It has two
distinct responsibilities:

1. **Game persistence** — stores analysis games sourced from PGN imports or finished matches,
   keyed per user. Exposes a REST API consumed directly by the client.

2. **Engine analysis relay** — accepts streaming analysis requests from the Analysis WebSocket
   Service via gRPC, calls the Engine service's `AnalyzePosition` endpoint, and streams the
   results back. It acts as a pure relay layer; no analysis logic lives here.

The client never connects to this service for streaming. All real-time analysis delivery goes
through the Analysis WebSocket Service (see [WebSocket Analysis Streaming](#websocket-analysis-streaming)).

## Technology

ASP.NET, consistent with Match Manager and Match Maker.

## Database

MongoDB, same cluster used by Match Manager. Analysis games are stored in a separate collection
(`analysis_games`). The analysis service holds no live match state.

## Data Model

An `AnalysisGame` document represents one saved game:

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `user_id` | string | Owner; all queries are scoped to this |
| `source` | enum `pgn \| match` | Origin of the game |
| `match_id` | string? | Populated when `source == match` |
| `moves` | string[] | Full move list in UCI notation, oldest first |
| `fens` | string[] | FEN after each move, parallel to `moves` |
| `pgn` | string | Raw PGN string (normalized on import) |
| `result` | string | PGN result token: `1-0`, `0-1`, `1/2-1/2`, or `*` |
| `white` | PlayerInfo | Name string for PGN imports; user/bot object for match imports |
| `black` | PlayerInfo | Same as white |
| `tags` | map | All PGN header key/value pairs |
| `created_at` | timestamp | |

`fens` is derived on import/save and stored so the client can navigate positions without
recomputing FENs from the move list on every request.

## REST API

See [`rest/analysis.md`](../maichess-api-contracts/rest/analysis.md).

Endpoints:

| Method | Path | Description |
|---|---|---|
| `GET` | `/games` | List the authenticated user's saved games |
| `GET` | `/games/{id}` | Get a single game with full move list and PGN |
| `POST` | `/games` | Import and save a game from PGN |
| `POST` | `/games/from-match/{match_id}` | Import a finished match as an analysis game |
| `DELETE` | `/games/{id}` | Delete a saved game |

## gRPC Interface

The analysis service exposes a single streaming RPC consumed by the Analysis WebSocket Service:

```
StreamPositionAnalysis(fen, bot_id, line_count) → stream PositionAnalysisUpdate
```

The service receives this call, opens a client-streaming connection to the Engine service's
`AnalyzePosition` RPC, and pipes each `AnalysisUpdate` back as a `PositionAnalysisUpdate`. The
stream terminates when the engine stops naturally or the caller cancels.

Proto: [`protos/analysis-service/v1/analysis.proto`](../maichess-api-contracts/protos/analysis-service/v1/analysis.proto)

## WebSocket Analysis Streaming

The client never calls the analysis service directly for streaming. The flow is:

```
Client (Next.js)
  │
  └─── WebSocket ──► Analysis WebSocket Service
                         └─── gRPC (StreamPositionAnalysis) ──► Analysis Service
                                                                    └─── gRPC (AnalyzePosition) ──► Engine Service
```

### WebSocket Protocol

The Analysis WebSocket Service manages the WebSocket lifecycle. The message schema is JSON.

**Endpoint:** `ws://analysis-ws-service/analysis`

**Auth:** Bearer token passed as query parameter `?token=<jwt>` on connection.

**Client → Server messages:**

```json
{ "type": "start_analysis", "fen": "rnbq.../...", "bot_id": "stockfish-5", "line_count": 3 }
```

```json
{ "type": "stop_analysis" }
```

`start_analysis` cancels any in-progress analysis for this connection before starting the new
one. Only one analysis stream per connection is active at a time.

**Server → Client messages:**

```json
{
  "type": "analysis_update",
  "depth": 14,
  "lines": [
    { "rank": 1, "evaluation_cp": 35, "moves": ["e2e4", "e7e5", "g1f3"] },
    { "rank": 2, "evaluation_cp": 12, "moves": ["d2d4", "d7d5", "c2c4"] }
  ]
}
```

```json
{ "type": "analysis_complete", "final_depth": 20 }
```

```json
{ "type": "error", "message": "engine unavailable" }
```

`analysis_update` is emitted once per completed search depth. `analysis_complete` is sent when
the engine terminates the stream naturally. On `stop_analysis` or connection close the stream is
cancelled silently (no `analysis_complete` is sent).

## Dependencies

| Direction | Service | Protocol | Purpose |
|---|---|---|---|
| Inbound | Client (via REST) | HTTP | Game persistence |
| Inbound | Analysis WebSocket Service | gRPC | Analysis streaming |
| Outbound | Engine Service | gRPC | `AnalyzePosition` |
| Outbound | Match Manager | gRPC | `GetMatch` (from-match import), `GetMatchPosition` (FEN reconstruction) |
| Outbound | Move Validator | gRPC | `GetLegalMoves`, `ValidateMove` (SAN→UCI conversion during PGN import) |
| Outbound | Database Service | gRPC | `Get`, `List`, `Insert`, `Delete` on `analysis_games` collection |

Auth is handled locally via JwtBearer middleware (same shared JWT key), not via gRPC calls to the Auth service.

## Authorization

- All REST operations are scoped to the authenticated user. One user cannot read or delete
  another user's saved games.
- `POST /games/from-match/{match_id}` additionally requires the user to have been a participant
  of the match (or the match to be public, if that concept is introduced later).
- The WebSocket connection is authenticated via the JWT query parameter on connect; the token is
  validated once at handshake time.

## Error Handling

- Invalid PGN on import returns `400` with a human-readable `error` field.
- If the Engine service is unavailable during a streaming session the analysis service closes the
  gRPC stream with status `UNAVAILABLE`; the WebSocket service translates this to an `error`
  message and may attempt reconnection per its own policy.
- The analysis service does not retry engine calls; retry policy is the caller's responsibility.
