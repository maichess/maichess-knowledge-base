# Insights & Spark analytics

> Status: ✅ **shipped** (task program [`tasks/implemented/insights/`](../../tasks/implemented/insights/README.md)).
> This ADR records the decisions; the per-task specs implement them.

## Problem

maichess has rich *live* game data (match-db, the `match.events.v1` log) but **no way
to analyze large historical corpora** of chess games — neither our own accumulated
matches nor public datasets. We want to download and analyze massive game collections
(starting with the monthly [Lichess database dumps](https://database.lichess.org/#standard_games))
and surface aggregate insights: most successful openings, most common endgames, most
common positions, and the "trickiest" positions (where players blunder or burn the
clock). Manual multi-game PGN upload must also work.

## Decision

A new **`maichess-insights-service`** plus a **Scala Apache Spark** batch module, with
an **S3-compatible MinIO data lake** and the **Kubeflow Spark Operator** running jobs on
the cluster. Results are materialized into Mongo and served over REST/gRPC; a new
`maichess-client` page consumes them.

### 1. Spark on Kubernetes via the Spark Operator
Jobs are `SparkApplication` custom resources reconciled by the kubeflow `spark-operator`
(Helm dependency, `maichess` namespace). The insights-service creates them on demand; a
`ScheduledSparkApplication` runs the monthly Lichess pull. This fits the existing
Helm + post-install-Job patterns better than ad-hoc `spark-submit` and gives a clean
control-plane→CRD seam. *Rejected:* plain K8s Jobs running `spark-submit` (no
distributed executors, capped at one pod); external/managed Spark (breaks the
self-hosted model, adds cost).

### 2. Spark jobs in Scala
Spark is Scala-native and the platform already has deep Scala expertise
(`maichess-engine-service`, `maichess-move-validator-service`). Scala 2.13 (Spark
connectors lag Scala 3), `sbt-assembly` fat jar. *Rejected:* PySpark — no Python service
exists today; a new toolchain to own.

### 3. Two artifacts, two languages — a deliberate split
- **Spark batch module (Scala)** does all heavy lifting and writes results to Mongo via
  the Spark MongoDB connector. It is the *only* thing that touches Spark/Parquet/MinIO.
- **`maichess-insights-service` (.NET, net10.0)** is the control plane + read API. It
  never reads Parquet — it reads the materialized `insights_*` collections through the
  existing **database-service gRPC** (conventions item 6), and submits/monitors
  `SparkApplication` CRDs via the C# Kubernetes client. This keeps the API tier
  consistent with the other .NET services (REST/gRPC, JWT, OTel, 100% coverage, Reqnroll).

*Rejected:* an all-Scala/ZIO service. The Scala/Spark surface stays narrowly scoped to
batch; results live in Mongo where the .NET API tier already knows how to read them.

### 4. Annotations-first analytics
Lichess dumps embed `[ECO]`/`[Opening]` headers and per-move `%eval`/`%clk` annotations,
so blunder, eval-swing ("tricky"), and think-time analysis are derived **without running
our engine**. Engine re-analysis (`engine-service` `AnalyzePosition` over top-N positions)
is a later, **opt-in, throttled, off-peak** enhancement — never the default — so insights
batch work cannot starve live bot moves / analysis sessions.

### 5. MinIO data lake; materialize to Mongo
No object storage existed before. MinIO (`insights-raw` → `insights-parsed` →
`insights-agg` buckets) holds downloaded dumps, uploads, and Parquet. Aggregates are
written to Mongo `insights_*` collections (via a dedicated **`insights-db`**
DatabaseService instance, conventions item 6) — that is the contract the read API and
client consume. Elasticsearch surfacing is optional / non-priority.

## Pipeline

```
lichess .pgn.zst / Chess.com / TWIC / manual PGN upload
  → (ingest job) download → decompress-once → parse → partitioned Parquet (insights-parsed)
  → (analysis jobs) opening / endgame / position / tricky → Mongo insights_* (+ insights-agg cache)
  → insights-service reads insights_* via database-service gRPC → REST/gRPC → client
```

The non-splittable `.zst` is the bottleneck (one core stream-decodes it), so ingestion
**decompresses once to a splittable form, then parses in parallel**; board replay (for
position/endgame FENs) is a separate parallel Spark stage over Parquet. See
[spark-and-minio](../operations/spark-and-minio.md) for placement, sizing, runtimes, and
live-platform isolation, and [insights-statistics](../domain/insights-statistics.md) for
the metric definitions.

## Pluggable sources

Ship **Lichess + manual PGN upload** first behind a `source adapter` interface so
**Chess.com monthly archives (JSON API)** and **TWIC weekly PGN** can be added later
without reworking the pipeline. (Other corpora: FICS games DB, PGN Mentor, Lumbra's Giga
Base, Lichess broadcasts.)

## Read API & client surface

The query endpoints (`GetTopOpenings` / `GetCommonEndgames` / `GetCommonPositions` /
`GetTrickyPositions` / `GetCorpusSummary` / `ListCorpora`, REST + gRPC) read the
materialized `insights_*` rows through database-service gRPC, fronted by a rebuildable
**Redis L1 cache** on the hot aggregates (top openings, summary, tricky), keyed by corpus
id + query params — the same `allkeys-lru`-tolerant seam analysis-service uses (noted in
[caching-and-read-models](caching-and-read-models.md)).

The `maichess-client` **Insights page** lives under the Tools menu at `/tools/insights`
(`requireUser`, all signed-in users). The landing view launches ingestions (Lichess-month
or PGN upload, with an optional rating/time/sample filter) and analysis runs, and polls
job status; `/tools/insights/{id}` is the per-corpus explorer — a summary header plus tabs
for openings (win/draw/loss bars + month-over-month trend), endgames (signature conversion
tendency), and common / tricky positions with board previews. All data flows through the
client REST proxy (`/api/insights/*` → `INSIGHTS_SERVICE_URL`); no contract change was
needed (the page consumes task 01's REST surface).
</content>
</invoke>
