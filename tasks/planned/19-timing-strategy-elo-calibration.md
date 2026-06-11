# 19 — Engine timing strategy ELO calibration via asymmetric arena series

> Read `services/maichess-engine-service/CLAUDE.md` and
> [engine-service.md](../../knowledge/services/engine-service.md) (if it exists) first.
> No contract changes. No new infrastructure.

## Goal

Determine empirically whether the `Proportional` and `Aggressive` timing strategies are actually
stronger than `Fixed` at each engine tier and time control, and update `BotRegistry` ELOs to
reflect measured strength rather than identical hardcoded estimates.

## Background

The timing code is correct: `BotMoveRequested.TimeLimitMs` carries the remaining game clock;
`computeMoveTime` dispatches to the strategy (Fixed uses a flat budget; Proportional divides
remaining time by a divisor; Aggressive is more front-loaded). A `blitz_proportional` bot at
div=30 gets ~10 s on move 1 of a 5-minute game vs 1 s for a Fixed `blitz` bot — materially
deeper search.

The problem: all three strategies for the same tier+speed have **identical hardcoded ELOs** in
`BotRegistry.scala`. In the current leaderboard/matchmaking these ELOs are only cosmetic, but
they will matter once matchmaking uses them for skill-based pairing.

**Why asymmetric matchups:** in a symmetric game (same strategy on both sides) any search-depth
advantage cancels out. You cannot measure strategy value from symmetric results. You must put
Fixed on one side and Proportional on the other, keep the engine tier identical, and count
wins/losses.

## Procedure

### Step 1 — design the series

For each tier (recommend starting with tier 2 — `EnhancedSearch` — as the cheapest meaningful
level), create two bot-arena Single or Matrix setups pitting the strategies head-to-head:

- `enhanced_blitz` (Fixed) vs `enhanced_blitz_proportional` (Proportional), 3+0, ≥50 games
- `enhanced_blitz_proportional` vs `enhanced_blitz_aggressive` (Aggressive), 3+0, ≥50 games

Run each at a longer time control too (e.g. rapid 10+0) to see whether the advantage grows with
game length (it should — Proportional's opening bonus is larger relative to a longer game).

### Step 2 — run in bot arena

Use the Single or Matrix setup with the standard FEN list (or a diverse custom FEN list to
reduce opening noise). For each pairing:
- Set up with `colorMode: AlternatingColors` so each FEN is played from both sides (removes
  white-advantage bias from the W/L count).
- Run at least 50 games per pairing direction.

### Step 3 — compute ELO deltas

From the W/L/D results, compute an empirical ELO difference using the standard formula:
```
ΔElo = -400 * log10(1/score - 1)
```
where `score = (wins + 0.5*draws) / total_games`.

Apply the delta to the Fixed ELO as the baseline; set Proportional and Aggressive ELOs relative
to it. Repeat per tier to check whether the delta is tier-independent.

### Step 4 — update `BotRegistry.scala`

Open `services/maichess-engine-service/src/main/scala/maichess/engine/domain/BotRegistry.scala`
and update the `elo` field for each bot in the affected tier × speed cells with the calibrated
values. Follow the existing pattern: `BotConfig(tier, speed, strategy, depth, elo)`.

Add a block comment above the first calibrated entry noting the calibration date and source
(e.g., "ELOs calibrated 2026-06-11 via 50-game asymmetric arena series, tier 2, 3+0 blitz").

### Step 5 — verify and document

- Run the existing engine-service test suite: `sbt test` — BotRegistry tests should still pass
  (they check structure, not ELO values).
- Record the raw W/L/D table, the computed deltas, and the methodology in this task's checklist
  or a companion `ELO_CALIBRATION.md` in `services/maichess-engine-service/`.
- Update `services/maichess-engine-service/CLAUDE.md` tier table to reflect the new ELOs.

## Tests

No new unit tests required — `BotRegistry` is a data file and its structural tests already
exist. The calibration evidence (W/L data) should be committed alongside the code change as
documentation.

## Checklist

- [ ] Tier 2 asymmetric series run (Fixed vs Proportional, Fixed vs Aggressive, 3+0, ≥50 games each).
- [ ] Same series at a longer time control (10+0 rapid) to check time-control dependence.
- [ ] ΔElo computed from W/L/D data.
- [ ] `BotRegistry.scala` ELO fields updated for calibrated tiers.
- [ ] Calibration date + methodology comment added to `BotRegistry.scala`.
- [ ] `services/maichess-engine-service/CLAUDE.md` tier table updated.
- [ ] `sbt test` passes.
- [ ] Repeat for remaining tiers (0, 1, 3, 4, 5) if tier 2 results are conclusive.
