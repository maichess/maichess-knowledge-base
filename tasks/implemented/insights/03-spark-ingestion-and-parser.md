# 03 — Spark ingestion + PGN parser

> Read [conventions.md](../../conventions.md),
> [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md),
> [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md), and
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md) first.
> Scala Spark module only (no contract change). Depends on the MinIO/Operator infra from task 02.

## Goal

Turn a source (a Lichess monthly dump, a manually uploaded multi-game PGN, or a future source) into
**partitioned Parquet** in MinIO `insights-parsed`, with the parsed schema from
[insights-statistics](../../../knowledge/domain/insights-statistics.md) (`games` + `plies`, including
`%eval`/`%clk`).

## What already exists (reuse it)

- PGN-format reference: Lichess dumps carry `[ECO]`/`[Opening]`/`[TimeControl]`/`[WhiteElo]` headers and
  per-move `%eval`/`%clk` comments. The existing analysis-service PGN parsing (regex header/movetext
  extraction in `AnalysisGameService.ParsePgn`) is a reference for the tag/movetext patterns — but this
  module is self-contained Scala (don't import the .NET code).
- Board representation for replay (FENs needed by position/endgame stats): **reuse the engine's pure
  `ChessPosition`** (`services/maichess-engine-service/.../chess/`) packaged as a small shared lib, or a
  vetted JVM chess library (chesslib). Opening / tricky / clock stats need **no** replay.

## Implementation

1. **Source adapter seam.** A small `SourceAdapter` interface (open a raw byte stream for a descriptor)
   with a `LichessMonthAdapter` (`https://database.lichess.org` standard month → `.pgn.zst`) and an
   `UploadedPgnAdapter` (a MinIO object key) implementation. Shape it so Chess.com/TWIC drop in later.
2. **Decompress-once, parse-parallel** (the [bottleneck mitigation](../../../knowledge/operations/spark-and-minio.md)):
   stream the `.zst` into `insights-raw`, **decompress once** to a splittable form in MinIO, then read
   it with a Spark text source split on the blank-line game delimiter so parsing parallelizes across all
   cores. Delete the decompressed scratch after Parquet is written.
3. **Pure PGN parser** (the bulk of the testable code, 100% covered): header map extraction, movetext
   tokenization, and `%eval` (centipawn, mate normalized) + `%clk` (→ ms, with increment handling)
   extraction per ply. Emit the `plies` rows.
4. **Optional board replay stage** (separate, parallel over Parquet): apply UCI/SAN moves with the pure
   board rep to fill `fen_before` per ply, only when position/endgame analysis is requested (it's the
   expensive part).
5. **Filters / sampling** applied at ingestion (rating band, time control, date slice, sample rate) so
   the **default corpus is filtered/sampled**; full-month is opt-in. Partition Parquet by
   `year_month` / `rating_band`.
6. **`IngestJob`** main wired into the Spark image (entry point invoked by the `SparkApplication` from
   task 05).

## Contract changes

None (the job is parameterized by the source/filter descriptor defined in task 01's proto, passed as
job args/config).

## Tests (mandatory)

- **Pure functions** (scalatest, 100%): header parse, movetext tokenization, `%eval`/`%clk` extraction,
  clock-delta/increment math, FEN-from-moves replay on a few known games, rating-band bucketing.
- **Transformations** against a `local[*]` `SparkSession` over small fixture PGNs (a handful of real
  Lichess-format games with `%eval`/`%clk`): assert the `games`/`plies` Parquet schema + row counts +
  partition columns. Network download path and MinIO I/O glue are excluded (mirror scoverage exclusions).

## Verify

- `sbt test` green. On staging, run `IngestJob` over a **small Lichess slice** (e.g. low-rating sample
  of one month) and a manual PGN upload → `insights-parsed` Parquet populates with correct partitions
  and the scratch is cleaned up.

## Knowledge base

- Update [insights-statistics](../../../knowledge/domain/insights-statistics.md) if the parsed schema
  changes. Mark task 03 `🟡` in [ROADMAP.md](../../ROADMAP.md).
