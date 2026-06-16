# Insights statistics — metric definitions

> Domain definitions for the insights program
> ([architecture/insights-and-spark](../architecture/insights-and-spark.md)). The Spark analysis
> jobs compute these; the read API and client surface them. Keep this doc and the job code in sync.

All metrics are computed **per corpus** (a corpus id identifies one analyzed dataset, e.g.
`lichess-2024-12`, `lichess-2024-12-blitz-1800plus`, `upload-{id}`) so results from different
sources/slices never blur together. Every job also records the **filter** that produced the corpus
(source, date slice, rating band, time control, sample rate).

## Parsed schema (Parquet, produced by ingestion)

- **games**: `corpus_id`, `game_id`, `result` (`1-0` / `0-1` / `1/2-1/2` / `*`), `eco`,
  `opening_name`, `white_elo`, `black_elo`, `time_control`, `termination`, `ply_count`,
  `year_month`, `rating_band`.
- **plies** (one row per half-move): `corpus_id`, `game_id`, `ply`, `side`, `san`, `uci`,
  `fen_before` (when board replay is run), `eval_cp` (from `%eval`, mate scores normalized),
  `clock_ms` (from `%clk`).

Rating band: bucket by min(or avg) Elo (e.g. `<1200`, `1200–1599`, `1600–1999`, `2000–2399`, `2400+`).

## Metrics

### Opening success (`insights_openings`)
Grouped by `eco` (and `opening_name`), optionally split by color, rating band, time control.
Reports game count, **win rate for White / Black / draw rate**, and a **popularity & success trend
over months**. No board replay needed — ECO/opening are in the PGN header.

### Common endgames (`insights_endgames`)
A position is an *endgame* when total piece count ≤ 7. Classify by **material signature** —
the multiset of non-king pieces per side written canonically, stronger side first (e.g. `KRPvKR`,
`KQvKR`, `KBNvK`); equal-strength signatures order the lexicographically smaller side string first
(ties → White is the "stronger" side). Report frequency and **result tendency** (how often the
stronger side converts to a win vs draw vs loss). Requires board replay to reach endgame FENs.

*Counting:* each `(game, signature, stronger-side)` reached is counted **once** — a long endgame
lingering on one signature does not inflate its frequency — and conversion uses the game's final
result. Frequency is therefore the number of games that reached the signature; `strongerSideWinRate`
/ `drawRate` / `strongerSideLossRate` are fractions of those games.

### Common positions (`insights_positions`)
Most-reached **normalized FENs**: the FEN with the halfmove-clock and fullmove-number fields
stripped (so transpositions and clock differences collapse), keeping piece placement, side to move,
castling, and en-passant. Optionally exclude positions still inside opening book (early plies, ply ≤
`bookPlies`) to surface interesting middlegame convergence. Requires board replay.

*Counting:* a position is counted **once per game** that reaches it; `reachCount` is the number of
distinct games, and `whiteWinRate` / `blackWinRate` / `drawRate` are over those games' results. A
`minReach` floor drops rarely-seen positions.

### Tricky positions (`insights_tricky`)
The intersection of two signals, both derived from annotations (no engine):
- **Blunder / eval-swing:** average **centipawn loss** of the move actually played from a position
  (drop in `eval_cp` from `fen_before` to after the played move, from White's perspective), and the
  **blunder probability** (fraction of times the move played loses more than a threshold, e.g. ≥300cp).
- **Think time:** time spent on the move, from consecutive `%clk` deltas for the *same side*. The
  eval *before* a move is the previous ply's `eval_cp`; the eval *after* is the move's own `eval_cp`.

"Trickiest" = positions reached often enough (`minSupport` played moves) with **both high average
centipawn loss and high think time** — i.e. where humans most often blunder under time pressure.
`support` is the number of moves observed from the position (one per ply, not per game). (Optional
later: fold in `engine-service` `AnalyzePosition` evals for top-N positions to corroborate the
`%eval` signal independently.)

> **Increment caveat.** The parsed `plies` schema (task 03) does not preserve the clock increment —
> the `TimeControl` header is bucketed to a category (`blitz`/`rapid`/…) at ingestion. The tricky
> job therefore computes think time from the raw same-side clock delta **without** adding the
> increment back, a uniform per-move understatement that does not affect the relative ranking. The
> pure `ThinkTime.spentMs` helper still models the increment for correctness and is unit-tested for
> it; restoring increment-accurate think time would require carrying raw `time_control` (or the
> parsed increment) on `games`/`plies`.

### Corpus summary (`insights_summary`)
Headline aggregates per corpus: total games, date range, rating distribution, draw rate, average
game length, **termination mix** (mate / resign / timeout / other), and **first-move popularity**.

### Bonus stats (cheap add-ons over the same Parquet)
Accuracy (centipawn-loss) vs rating, time-trouble blunder rate, draw rate by opening / time control,
color advantage by rating band — all groupings over the `plies` / `games` tables, surfaced as the
program matures.

## Job catalog / status (`insights_jobs`)
Each ingestion/analysis run is recorded with its corpus id, source descriptor, filter, status
(`pending` / `running` / `succeeded` / `failed`), timestamps, and the `SparkApplication` name — read
by the control plane to track and list jobs.
</content>
