# 03 — Glicko-2 rating system (chess.com-style, base 400)

**Goal:** Replace the static `elo` with a real rating computed from a player's
recent games, using **Glicko-2** (the algorithm chess.com uses). Seed/base rating
is **400**. Builds directly on the `RecordMatchResult` path from `02`.

> Read `conventions.md` first. Depends on `02`.

## Background to read first

- `02`'s additions: user-service `RecordMatchResult` and the match-manager
  match-end fan-out. This prompt extends `RecordMatchResult`; it does **not** add a
  new mutation entry point.
- user-service: `UsersService.cs` (Struct-backed user record via database-service),
  `protos/user-service/v1/users.proto`.
- Glicko-2 reference: Glickman's "Example of the Glicko-2 system" (the canonical
  spec). Implement it faithfully; don't approximate.

## Algorithm decisions (implement exactly)

- **Glicko-2 per player:** rating `r` (display scale), rating deviation `RD`, and
  volatility `σ`. Internal Glicko-2 scale uses `μ = (r − 1500)/173.7178` and
  `φ = RD/173.7178`; convert back for storage/display.
- **Seed/base = 400.** New players start at `r = 400`, `RD = 350` (high, fast-moving
  for new accounts), `σ = 0.06`, `τ` (system constant) `= 0.5`. Document these in
  the ADR; they are the only knobs.
- **"Recent games" rating period:** Glicko-2 updates over a *rating period* (a batch
  of recent games), not strictly per-game. Implement an incremental period update:
  accumulate a player's games since their last update and apply the period update,
  OR run a per-game update with a single-opponent period (simplest correct form).
  Choose the single-opponent-per-update form so it slots into the per-result
  `RecordMatchResult` call; document the trade-off in the ADR.
- **Display `elo` = round(r).** Keep the `User.elo` field as the rounded rating so
  the existing client (`Nav`, Profile) needs no change to *read* it.

## Contract changes (`users.proto` + `rest/users.md`)

- Add to `User`: `double rating = 8;` (the unrounded Glicko-2 `r`),
  `double rating_deviation = 9;`, `double volatility = 10;`. Keep `elo` as the
  rounded display value derived from `rating`.
- `RecordMatchResult` already carries outcome + opponent context; ensure it includes
  the **opponent's current rating + RD** (add `double opponent_rating = 3;`,
  `double opponent_rd = 4;` to `RecordMatchResultRequest`) so user-service can run the
  Glicko-2 update without a second lookup. match-manager supplies these from the
  opponent's profile at match end.
- Expose the new fields in the REST user response.
- Versioning handoff per `00`: publish a new tagged contracts version; bump
  `Maichess.PlatformProtos` across all services.

## Service changes

### user-service
- New pure module `Rating/Glicko2.cs` (no I/O): functions for scale conversions,
  the `g(φ)` and `E(μ, μ_j, φ_j)` helpers, the volatility iteration (Illinois /
  regula-falsi as in the spec), and the period update returning new `(r, RD, σ)`.
  This is the heart of coverage — unit-test it exhaustively against the published
  worked example and edge cases (win/loss/draw, provisional high-RD, convergence of
  the volatility solver). **100% line/branch/method.**
- `CreateUserAsync`: seed `rating = 400`, `rating_deviation = 350`,
  `volatility = 0.06`, `elo = 400` (replace the current `1200` default).
- `RecordMatchResult`: load the user's `(r, RD, σ)`, run `Glicko2.Update` with the
  opponent rating/RD + outcome score (1 / 0.5 / 0), persist new `(r, RD, σ)` and
  `elo = round(r)`, and still increment W/L/D from `02`.
- Tests: integration-level tests that a recorded result moves rating in the right
  direction and persists all fields; plus the exhaustive `Glicko2` unit tests.
  Update Stryker config to include the new module (it is core logic — **not**
  excluded).

### match-manager
- At match end, before calling `RecordMatchResult`, fetch each human's opponent
  rating + RD (from the opponent's user profile, or from the bot's configured rating
  for human-vs-bot) and pass them in. Add tests for the new wiring. Bots: use the
  bot's `elo` from the engine `Bot` metadata as the opponent rating with a low RD
  (treat bots as having an established rating — document the chosen bot RD in the ADR).

## Client changes

- Minimal: `elo` already displays. Optionally show `± RD` (provisional indicator
  when RD is high) on the Profile stats card and a tooltip explaining the rating is
  Glicko-2 based. Update `lib/models/user.ts` with the new fields if surfaced.

## Verification

- `cd services/maichess-user-service && dotnet test -p:CollectCoverage=true` → 100%,
  with the `Glicko2` module fully covered; `dotnet stryker` → review any surviving
  mutants in the rating math.
- match-manager tests green at 100%.
- Manual: play several games against bots of differing strength → rating moves
  plausibly (toward bot strength), RD shrinks as games accumulate, new accounts
  start at 400.

## Knowledge base

Add an ADR `maichess-knowledge-base/rating-glicko2.md`: why Glicko-2 (chess.com
parity), the constants (`r₀=400, RD₀=350, σ₀=0.06, τ=0.5`), the per-result update
form chosen and its trade-off vs batched rating periods, and how bot opponents are
rated.
