# 16 — Lichess compatibility for the tournament bridge

> Read [conventions.md](../conventions.md) and
> [external-games.md](../../knowledge/domain/external-games.md) first.
> **Supersedes the aborted prompt 06.** The bridge already exists and works for the NowChess
> **tournament-server**; this task adds **Lichess** as a second provider behind the same
> engine-drives/we-mirror model. Scope: `maichess-tournament-bridge-service` only (plus its REST
> contract + a small client entry point). We never modify `tournament-server/`.

## Goal

Let a user register a maichess bot to play a **Lichess** game (or accept a Lichess challenge), with
the maichess **Engine driving the moves**, mirroring the game into match-db as a read-only
`external` match so it shows in Watch and Past Matches tagged "external" — exactly as
tournament-server games already do.

## What already exists (reuse it)

The bridge (`maichess-tournament-bridge-service`) already implements the hard parts, provider-agnostic:
- **`GameDriver`** — pure logic: given a game event (position/turn/clock) + our color + bot config,
  decide the action (SubmitMove / SyncMatch / FinalizeMatch). **Provider-independent — reuse as-is.**
- **`TournamentOrchestrator` / IO adapter** — opens the provider stream, calls Engine `GetBestMove`,
  submits moves, and mirrors via match-manager `CreateMatch(source=EXTERNAL)` + `SyncExternalMatch`.
- **External-match semantics** — unrated (never calls `RecordMatchResult`), SSE/socket broadcast via
  the normal mirror path, `external_ref` back-link. **Unchanged.**

The gap is purely the **provider protocol adapter**: tournament-server NDJSON+JWT is wired; Lichess
is not.

## Lichess Bot API (the new provider)

Reference only (nothing to generate): https://lichess.org/api#tag/Bot
- Account must be upgraded to a **bot** account; auth is the user's Lichess **bot OAuth token** (bearer).
- `GET /api/stream/event` — incoming challenges/game-starts (NDJSON).
- `GET /api/bot/game/stream/{gameId}` — per-game state stream (NDJSON: `gameFull` then `gameState`).
- `POST /api/bot/game/{gameId}/move/{uci}` — submit a move.
- (Optional) `POST /api/challenge/{user}` / `POST /api/bot/game/{gameId}/abort|resign`.

Clock: Lichess `gameState` carries `wtime`/`btime`/`winc`/`binc` in **ms** (match-db is already ms —
no `*1000` conversion needed, unlike tournament-server's seconds).

## Implementation

1. **Provider abstraction.** Introduce a small `IExternalProvider` (or extend the existing client
   seam) with: open-event-stream, open-game-stream, submit-move, and parse-event→domain. Make the
   existing tournament-server client one implementation; add a `LichessProvider` implementation.
   Keep `GameDriver` untouched — only the adapter differs.
2. **Lichess event parsing (pure, 100% covered).** Map `gameFull`/`gameState`/`chatLine` NDJSON to
   the bridge's existing game-event domain type; derive FEN/turn from the moves list, our color, and
   clocks. This is the bulk of the testable code — mirror the existing tournament-server parser tests.
3. **Lichess IO adapter (`[ExcludeFromCodeCoverage]`).** A typed `HttpClient` wrapper for the three
   endpoints with bearer auth; behind the provider interface so the orchestration that uses it is
   tested with a fake.
4. **Registration.** Add `POST /external/lichess` to the bridge REST (body `{ bot_id, lichess_token,
   game_id }`, or a challenge id). Validate bot existence + token presence, create the external match
   via match-manager, and start a per-game bridge driver. Return the maichess `match_id` so the game
   is immediately watchable.
5. **Client (optional, small).** A "Play on Lichess" entry in the Dev section that calls the
   registration endpoint; external matches already render via the existing read-only viewer + badge.

## Contract changes

- `rest/tournament-bridge.md`: add the `POST /external/lichess` endpoint.
- No proto change expected — `CreateMatch(source=EXTERNAL, external_provider="lichess", external_ref)`
  and `SyncExternalMatch` already exist. If a new field is genuinely needed, follow the contract
  versioning handoff in [conventions.md](../conventions.md).

## Tests (mandatory)

- Lichess NDJSON → domain parsing, turn detection, clock mapping (ms, no conversion), provider
  URL/auth construction — pure, **100%** line/branch/method.
- Registration service: valid/invalid bot, missing token, provider routing, error mapping.
- IO adapter excluded from coverage (live HTTP), tested via a faked provider behind `IExternalProvider`.
- `dotnet test -p:CollectCoverage=true`; `dotnet stryker` to audit the new parser/turn-detection mutants.

## Verify

- Manual (Lichess): with a real Lichess bot token + an open game, `POST /external/lichess` → the game
  appears in Watch, the Engine plays our moves on Lichess, moves mirror live, and it lands in Past
  Matches tagged External and **unrated**.
- Regression: tournament-server registration/bridging still works unchanged.

## Knowledge base

Update [external-games.md](../../knowledge/domain/external-games.md): flip its status to note Lichess
is supported, and document the `IExternalProvider` seam (tournament-server + Lichess implementations).
