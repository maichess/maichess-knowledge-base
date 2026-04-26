# Maichess Structure

Maichess is a web-based, complete chess application featuring variable game rules, online multiplayer and bot integration.

## Microservices

It is built as a microservices architecture deployed via docker compose. The [client](##client) communicates with [Match Maker](##match-maker), [Match Manager](##match-manager), [Auth](##auth) and [User](##user) services via HTTP, microservices communicate with each other via gRPC. Databases are accessed via database services that handle db connections and queries. 

### Client

*Next.js, Tailwind.*

Only user-facing service. 

Communicates with [Match Maker](##match-maker) to initate sessions and then with [Match Manager](##match-manager) to play a session.

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
- `match-db` — MongoDB, serving [Match Manager](##match-manager).

## Databases

Two main databases are involved.

- **Postgres** for storing user data that is queried by [User](##user) and [Auth](##auth).
- **MongoDB** for storing game states queried by [Match Manager](#match-manager).