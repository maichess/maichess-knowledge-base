# External Games

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
= "tournament-server")` and updated via `SyncExternalMatch`. This path:

- **Skips** move validation (the provider already validated)
- **Skips** bot-move scheduling (the bridge drives moves externally)
- **Does** broadcast socket events (`MoveMadeEvent`, `MatchEndedEvent`) so Watch
  spectators get real-time updates through the existing pipeline
- **Does not** call `RecordMatchResult` — external games are **unrated** and do
  not affect any player's W/L/D or Glicko-2 rating

Each mirrored match carries an `external_ref` linking back to the provider's
game ID, enabling the client to cross-reference tournament context.

## Provider contract

`tournament-server/` is treated as an external, read-only contract. The only
permitted modification is adding auth registration endpoints (`POST
/api/auth/register`) so the bridge can obtain JWTs programmatically. All other
tournament-server behavior is consumed as-is via its OpenAPI spec.

## Where it lives

- **Knowledge base:** this document
- **Contracts:** `Player.external_name` + `CreateMatchRequest.source/external_provider/external_ref`
  + `Match.external_ref` + `SyncExternalMatch` RPC in `protos/match-manager-service/v1/matches.proto`;
  bridge REST spec in `rest/tournament-bridge.md`
- **Services:** `maichess-tournament-bridge-service` (new), changes in `maichess-match-manager-service`
- **Client:** `/tournaments` routes, external match badges in Watch/Past Matches
