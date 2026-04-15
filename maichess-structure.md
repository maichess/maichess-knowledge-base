# Maichess Structure

Maichess is a web-based, complete chess application featuring variable game rules, online multiplayer and bot integration.

## Microservices

It is built as a microservices architecture deployed via docker compose. The [client](##client) communicates with [Match Maker](##match-maker), [Match Manager](##match-manager), [Auth](##auth) and [User](##user) services via HTTP, microservices communicate with each other via gRPC.

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

Accepts a full game state and a bot identifier, then calculates the next following move based on the specified bot's logic. 

Once a move has been calculated, calls back to [Match Manager](##match-manager) to execute the move.

### User 

*ASP.NET*

Handles user profile creation and management.

### Auth

*Express.js*

Accepts login and registration requests. 

If it is a login request, uses read-only db access to verify credentials, if registration requests creation of a new user from the [User service](##user). 

Issues JWT tokens if the login or registration was successful.

## Databases

Two main databases are involved.

- **Postgres** for storing user data that is queried by [User](##user) and [Auth](##auth).
- **MongoDB** for storing game states queried by [Match Manager](#match-manager).