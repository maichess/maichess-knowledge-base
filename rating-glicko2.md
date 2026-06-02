# Player ratings: Glicko-2

## Decision

maichess rates players with **Glicko-2** (the system chess.com uses), not a
static Elo. Each player carries three rating fields:

- `rating` (`r`) — the rating on the display scale (unrounded).
- `rating_deviation` (`RD`) — uncertainty; high for new/inactive players,
  shrinking as games accumulate.
- `volatility` (`σ`) — how erratic recent results have been.

`User.elo` is kept as **`round(rating)`** so existing clients (Nav, Profile)
read a single integer strength without change.

## Constants (the only knobs)

| Constant | Value | Meaning |
|---|---|---|
| `r₀`  | **400**  | seed/base rating for new accounts |
| `RD₀` | **350**  | seed deviation (high → new ratings move fast) |
| `σ₀`  | **0.06** | seed volatility |
| `τ`   | **0.5**  | system constant constraining volatility change |

The internal Glicko-2 scale uses the canonical transform
`μ = (r − 1500)/173.7178`, `φ = RD/173.7178` (`173.7178 = 400/ln 10`). The
additive anchor (1500) is the standard Glickman constant and is purely a
zero-point: because every rating is converted with the same anchor, only rating
*differences* affect the result, so seeding new players at a display rating of
400 instead of 1500 is self-consistent and changes nothing in the math. We seed
at 400 only because that is the desired displayed starting strength.

## Update form: per-result, single-opponent rating period

Glicko-2 is defined over a **rating period** (a batch of games). We chose the
**single-opponent-per-update** form: each `RecordMatchResult` call applies a
rating period containing exactly one game. This slots directly into the existing
per-result fan-out from Match Manager with no batching infrastructure.

- **Trade-off vs. batched periods:** applying games one at a time is a documented
  approximation of the batched ideal — `RD`/`σ` evolve slightly differently than
  if a day's games were processed together, and ordering matters marginally. In
  exchange we avoid a scheduler and a pending-games store, and ratings update
  immediately at match end. For a casual platform this is the right trade.
- The pure module (`Rating/Glicko2.cs`) is nonetheless written against a **list**
  of games (the general algorithm), so it is unit-tested against Glickman's
  published 3-opponent worked example; the caller simply passes a one-element
  list. The empty-list path implements the idle-period RD inflation for
  completeness.

## How opponents are rated

`RecordMatchResultRequest` carries the **opponent's `rating` + `RD`** so
user-service needs no second lookup. Match Manager supplies them at match end,
snapshotting **both** sides' pre-match ratings *before* recording either result
(so a human-vs-human pair is each rated against the other's prior rating, not one
already updated by the same fan-out):

- **Human opponent:** read from the opponent's user profile (`GetUser` →
  `rating`, `rating_deviation`).
- **Bot opponent:** the bot's engine-configured `elo` as the rating, with a
  fixed low deviation of **`RD = 50`** — bots are treated as having an
  established rating. An unknown bot id falls back to rating `0`.

Bot-vs-bot games still record nothing (no human participant), so they never move
any player's rating, consistent with [[match-history-and-stats]].

## Where it lives

- **Contract** (`Maichess.PlatformProtos` v0.3.5):
  - `User.rating` (8), `rating_deviation` (9), `volatility` (10);
    `RecordMatchResultRequest.opponent_rating` (3), `opponent_rd` (4) in
    `protos/user-service/v1/users.proto`. New fields exposed in `rest/users.md`.
- **user-service:** pure `Rating/Glicko2.cs` (no I/O — scale conversions, `g(φ)`,
  `E`, the Illinois volatility solver, the period update). `CreateUser` seeds
  `r=400, RD=350, σ=0.06, elo=400`. `RecordMatchResult` loads `(r, RD, σ)`, runs
  the update with score `1 / 0.5 / 0`, and persists the new `(r, RD, σ)` plus
  `elo = round(r)` alongside the W/L/D increment. Records written before these
  fields existed fall back to elo-derived state. The module is core logic — **not**
  excluded from coverage or mutation testing.
- **client:** `lib/models/user.ts` gains `rating`/`rating_deviation`/`volatility`
  and an `isProvisionalRating` helper; the Profile stats card shows `± RD` and a
  provisional marker while `RD` is high.
