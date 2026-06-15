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
`KQvKR`, `KBNvK`). Report frequency and **result tendency** (how often the side-to-move/stronger
side converts to a win vs draw). Requires board replay to reach endgame FENs.

### Common positions (`insights_positions`)
Most-reached **normalized FENs**: the FEN with the halfmove-clock and fullmove-number fields
stripped (so transpositions and clock differences collapse), keeping piece placement, side to move,
castling, and en-passant. Optionally exclude positions still inside opening book (early plies) to
surface interesting middlegame convergence. Requires board replay.

### Tricky positions (`insights_tricky`)
The intersection of two signals, both derived from annotations (no engine):
- **Blunder / eval-swing:** average **centipawn loss** of the move actually played from a position
  (drop in `eval_cp` from `fen_before` to after the played move, from White's perspective), and the
  **blunder probability** (fraction of times the move played loses more than a threshold, e.g. ≥300cp).
- **Think time:** time spent on the move, from consecutive `%clk` deltas (accounting for increment).

"Trickiest" = positions reached often enough (min support) with **both high average centipawn loss
and high think time** — i.e. where humans most often blunder under time pressure. (Optional later:
fold in `engine-service` `AnalyzePosition` evals for top-N positions to corroborate the `%eval`
signal independently.)

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
