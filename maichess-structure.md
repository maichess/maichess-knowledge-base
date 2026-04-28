# Maichess Structure

Maichess is a web-based, complete chess application featuring variable game rules, online multiplayer and bot integration.

## Microservices

It is built as a microservices architecture deployed via docker compose. The [client](##client) communicates with [Match Maker](##match-maker), [Match Manager](##match-manager), [Auth](##auth), [User](##user), and [Analysis](##analysis) services via HTTP. Microservices communicate with each other via gRPC. Databases are accessed via database services that handle db connections and queries.

### Client

*Next.js, Tailwind.*

Only user-facing service.

Communicates with [Match Maker](##match-maker) to initiate sessions and then with [Match Manager](##match-manager) to play a session. Connects to [Socket Service](##socket-service) via socket.io for real-time match events and analysis updates. Communicates with [Analysis](##analysis) for game persistence and analysis session control.

### Match Maker

*ASP.NET*

Handles player session initialization requests.

If players are looking for human opponents it matches them to other waiting players. If there are enough waiting players, attempts to match based on skill level.

If players are looking for bot opponents it directly spins up an instance.

Once an opponent has been found, delegates to [Match Manager](##match-manager) and returns session identifier to the user.

### Match Manager

*ASP.NET*

Instantiates games and accepts move requests.

Defers to [Move Validator](##move-validator) to confirm legality of moves and stores state updates in the database.

If a bot is expected to move next after a ply, triggers a bot move from the [Engine](##engine).

### Move Validator

*Scala ZIO*

Accepts a full game state (FEN) and a proposed move and checks for legality and win conditions.

Exposes four RPCs:

- **`ValidateMove`** — validates a move in UCI notation; returns the resulting FEN and game result.
- **`ValidateMoveSan`** — validates a move in SAN notation; returns the resulting FEN, game result, and the canonical UCI form of the move.
- **`GetLegalMoves`** — returns all legal UCI moves for a given FEN.
- **`GetLegalMovesSan`** — returns all legal moves as `(uci, san)` pairs for a given FEN.

`ValidateMove` and `GetLegalMoves` are called by Match Manager. `ValidateMoveSan` and `GetLegalMovesSan` are called by Analysis. All four variants share the same internal chess logic.

### Engine

*Scala ZIO*

Exposes two capabilities over gRPC:

**Move calculation (`GetBestMove`)** — accepts a board position (FEN), a bot identifier, and an optional time limit in milliseconds. Returns the best move in UCI notation plus a centipawn evaluation.

**Position analysis (`AnalyzePosition`)** — server-streaming endpoint. Accepts a position (FEN), a bot identifier, and the number of lines (`line_count`) to evaluate simultaneously. Streams one `AnalysisUpdate` per completed search depth; each update contains all requested lines ranked by evaluation. The engine decides internally when to stop streaming. Callers may cancel the stream at any time.

### Analysis

*ASP.NET*

Manages saved analysis games and analysis sessions.

Exposes a REST API for game persistence: users can import games from PGN, copy a finished match
from match-db, or create a game from an arbitrary FEN. Each saved game stores the full move list
and the FEN after every move so the client can navigate positions.

For real-time analysis, the client creates an analysis session (REST) and starts analysis. The
analysis service calls the Engine's `AnalyzePosition` endpoint and pushes each depth update to
the client via the Socket Service (`Socket.EmitEvent`) using `analysis_update` events. Analysis
results for the configured default bot are cached in the database and served transparently before
live engine output — the client receives the same event stream regardless of whether data comes
from cache or the engine.

The client receives analysis events on the same socket.io connection used for match events; no
separate WebSocket connection is needed.

See [analysis-service.md](analysis-service.md) for the full data model, protocol, and dependency map.

### Socket Service

*Node.js / socket.io*

Real-time event push gateway. Maintains persistent socket.io connections from clients and exposes
a gRPC `EmitEvent` endpoint so other services can push events to a connected user without knowing
anything about WebSocket internals.

Authenticates connecting clients by calling `Auth.ValidateToken` via gRPC. Services (Match
Manager, Match Maker, Analysis) call `Socket.EmitEvent` to deliver events such as `move_made`,
`match_ended`, `matched`, and `analysis_update`.

### User

*ASP.NET*

Handles user profile creation and management.

### Auth

*Express.js*

Accepts login and registration requests.

If it is a login request, uses read-only db access to verify credentials. If registration, requests creation of a new user from the [User service](##user).

Issues JWT tokens if the login or registration was successful.

### Database Service

Handles all database access for a single configured domain. Exposes a generic CRUD interface over gRPC and translates requests into DB-specific queries internally, keeping all DB-specific code isolated to adapter implementations.

Two instances are deployed:

- `user-db` — PostgreSQL, serving [User](##user) and [Auth](##auth). The Auth service is restricted to read-only operations via instance configuration.
- `match-db` — MongoDB, serving [Match Manager](##match-manager) and [Analysis](##analysis).

## Databases

Two main databases are involved.

- **Postgres** for storing user data that is queried by [User](##user) and [Auth](##auth).
- **MongoDB** for storing game states queried by [Match Manager](#match-manager) and analysis games / analysis cache queried by [Analysis](#analysis).
