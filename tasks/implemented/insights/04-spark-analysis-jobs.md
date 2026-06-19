# 04 — Spark analysis jobs

> Read [conventions.md](../../conventions.md),
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md) (metric definitions —
> implement exactly), and [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md)
> first. Scala Spark module. Depends on the contract (01) and parsed Parquet (03).

## Goal

Compute the insight metrics over `insights-parsed` Parquet and **materialize them into the
`insights_*` Mongo collections** (`insights-db`) via the Spark Mongo connector, plus cache aggregate
Parquet in `insights-agg` for cheap re-aggregation.

## What already exists (reuse it)

- The parsed `games`/`plies` schema and every metric's precise definition in
  [insights-statistics](../../../knowledge/domain/insights-statistics.md) — opening success, endgame
  material signature, normalized-FEN position frequency, "tricky" = eval-loss ∩ think-time, summary,
  bonus stats, and the `insights_jobs` record.
- The Spark MongoDB connector (bundled in the task-02 image) for the write path — the one place the
  Spark module writes to a datastore directly (documented exception in the ADR; the .NET service still
  reads via database-service).

## Implementation

One job (or one Spark app with sub-stages) per metric, each writing to its collection, all keyed by
`corpus_id` and recording the corpus filter:

1. **`OpeningStatsJob`** → `insights_openings`: group by `eco`/`opening_name` (× color, rating band,
   time control), win/draw/loss rates, popularity & success trend over months. No board replay.
2. **`EndgameStatsJob`** → `insights_endgames`: over replayed FENs with ≤7 pieces, canonical **material
   signature** (e.g. `KRPvKR`), frequency + conversion tendency.
3. **`PositionFrequencyJob`** → `insights_positions`: most-reached **normalized FENs** (strip
   halfmove-clock + fullmove-number), optionally excluding early-book plies.
4. **`TrickyPositionJob`** → `insights_tricky`: per position, average **centipawn loss** + **blunder
   probability** (from `%eval`) and **think time** (from `%clk` deltas); rank by the intersection with a
   minimum support threshold.
5. **`SummaryJob`** → `insights_summary`: totals, date range, rating distribution, draw rate, average
   length, termination mix, first-move popularity. Add **bonus stats** here as the program matures.
6. **Job-record write** to `insights_jobs` (status/timestamps/SparkApplication name) — the control
   plane (task 05) reads this; the Spark side updates it on completion/failure.

## Contract changes

None.

## Tests (mandatory)

- **Pure aggregation logic** (scalatest): material-signature canonicalization, FEN normalization,
  centipawn-loss / blunder-threshold, clock-delta think time, rating-band grouping.
- **Jobs** against a `local[*]` `SparkSession` over fixture Parquet (built from the task-03 fixtures):
  assert a known opening's win rate, a known endgame signature count, a known normalized-FEN frequency,
  and a known `%eval` blunder are computed correctly. The Mongo-connector write is exercised via a
  small embedded/local Mongo or mocked sink; the live write glue is excluded (mirror scoverage).

## Verify

- `sbt test` green. On staging, run the jobs over the small slice ingested in task 03 → `insights_*`
  collections populate with sane values; spot-check a top opening and a tricky position by hand.

## Knowledge base

- Keep [insights-statistics](../../../knowledge/domain/insights-statistics.md) in sync with what each
  job actually computes. Mark task 04 `🟡` in [ROADMAP.md](../../ROADMAP.md).
