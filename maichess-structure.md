# Maichess Structure

Maichess is a web-based, complete chess application featuring variable game rules, online multiplayer and bot integration.

## Microservices

It is built as a microservices architecture deployed via docker compose. The [client](##client) communicates with [Match Maker](##match-maker), [Match Manager](##match-manager), [Auth](##auth) and [User](##user) services via HTTP, microservices communicate with each other via gRPC. Databases are accessed via database services that handle db connections and queries. 

### Client

*Next.js, Tailwind.*

Only user-facing service.

Communicates with [Match Maker](##match-maker) to initiate sessions and then with [Match Manager](##match-manager) to play a session. Connects to [Socket Service](##socket-service) via socket.io for real-time match events. Connects to [Analysis WebSocket Service](##analysis-websocket-service) via plain WebSocket for real-time analysis delivery. Communicates with [Analysis](##analysis) for game persistence.

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

*Scala Zio*

Accepts a full game state and proposed next move and checks for legality and win conditions.

### Engine

*Scala Zio*

Exposes two capabilities over gRPC:

**Move calculation (`GetBestMove`)** — accepts a board position (FEN), a bot identifier, and an optional time limit in milliseconds. Returns the best move in UCI notation plus a centipawn evaluation. If no time limit is provided, the engine searches without a time constraint. Once a move has been calculated, calls back to [Match Manager](##match-manager) to execute the move.

**Position analysis (`AnalyzePosition`)** — server-streaming endpoint. Accepts a position (FEN), a bot identifier, and the number of lines (`line_count`) to evaluate simultaneously (equivalent to UCI `MultiPV`). Streams one `AnalysisUpdate` per completed search depth; each update contains all requested lines ranked by evaluation, each with its full principal variation (move sequence). The engine decides internally when to stop streaming (depth cap or no further improvement). Callers may cancel the stream at any time via standard gRPC cancellation.

### Analysis

*ASP.NET*

Manages saved analysis games and relays engine analysis streams.

Exposes a REST API for game persistence: users can import games from PGN or copy a finished
match, save them, and retrieve them later. Each saved game stores the full move list and the FEN
after every move so the client can navigate positions without recomputing them.

For real-time analysis the service exposes a gRPC streaming endpoint consumed by the Analysis
WebSocket Service. When that service opens a `StreamPositionAnalysis` call, the analysis service
proxies it to the Engine service's `AnalyzePosition` endpoint and streams the depth-by-depth
updates back. The client communicates with the Analysis WebSocket Service over WebSocket; it
never calls the analysis service directly for streaming.

See [analysis-service.md](analysis-service.md) for the full data model, protocol, and dependency map.

### Socket Service

*Node.js / socket.io*

Real-time event push gateway. Maintains persistent socket.io connections from clients and exposes
a gRPC `EmitEvent` endpoint so other services can push events to a connected user without knowing
anything about WebSocket internals.

Authenticates connecting clients by calling `Auth.ValidateToken` via gRPC. Other services
(Match Manager, Match Maker) call `Socket.EmitEvent` to deliver events such as `move_made`,
`match_ended`, and `matched`.

### Analysis WebSocket Service

*(Planned — not yet implemented)*

Manages persistent plain WebSocket connections from the client for real-time analysis delivery.
Endpoint: `ws://analysis-ws-service/analysis`. Authenticates via JWT query parameter on
connection, then calls the Analysis Service via gRPC to obtain an analysis stream and forwards
each depth update as a JSON WebSocket message. Only one analysis stream per connection is active
at a time; a `start_analysis` message implicitly cancels any running stream before starting the
new one.

See [analysis-service.md](analysis-service.md) for the full WebSocket protocol.

### User 

*ASP.NET*

Handles user profile creation and management.

### Auth

*Express.js*

Accepts login and registration requests. 

If it is a login request, uses read-only db access to verify credentials, if registration requests creation of a new user from the [User service](##user). 

Issues JWT tokens if the login or registration was successful.

### Database Service

Handles all database access for a single configured domain. Exposes a generic CRUD interface over gRPC and translates requests into DB-specific queries internally, keeping all DB-specific code isolated to adapter implementations.

Two instances are deployed:

- `user-db` — PostgreSQL, serving [User](##user) and [Auth](##auth). The Auth service is restricted to read-only operations via instance configuration.
- `match-db` — MongoDB, serving [Match Manager](##match-manager) and [Analysis](##analysis).

## Databases

Two main databases are involved.

- **Postgres** for storing user data that is queried by [User](##user) and [Auth](##auth).
- **MongoDB** for storing game states queried by [Match Manager](#match-manager) and analysis games queried by [Analysis](#analysis).