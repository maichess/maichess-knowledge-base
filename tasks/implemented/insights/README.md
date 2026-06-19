# Insights & Spark analytics — task program

> Status: ✅ **shipped.** `01` contract published (`v0.14.0`), `02` deploy infra landed, `03`
> ingestion/parser shipped, `04` analysis jobs shipped (Scala module, 73 tests green; the
> `local[*]` Spark suites run on Java 17), `05` control plane shipped — compiles against
> `Maichess.PlatformProtos 0.14.0`, **106 tests green at 100% line/branch/method**, `06` query API
> shipped (REST + gRPC + Redis L1), and `07` client Insights page shipped (`npm run build` +
> `npm run lint` clean). The whole pipeline (`01`–`07`) is delivered end to end. An ordered set of
> self-contained tasks that add a new
> **`maichess-insights-service`** + a **Scala Apache Spark** batch module to analyze massive
> historical chess corpora (starting with the monthly
> [Lichess database dumps](https://database.lichess.org/#standard_games)). Read the design first:
> [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md) (decisions,
> pipeline), [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md) (placement,
> sizing, runtimes, isolation), and
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md) (metric definitions).

**Run them in order** (dependencies below). Each `NN-*.md` is one working session. The shared rules in
[../../conventions.md](../../conventions.md) apply to all of them.

## Guiding decisions (already made — implement, don't re-litigate)

See the [ADR](../../../knowledge/architecture/insights-and-spark.md) for the full rationale:

1. **Spark on k8s via the Kubeflow Spark Operator** (`SparkApplication` CRDs), not ad-hoc
   `spark-submit` and not managed/external Spark.
2. **Spark jobs in Scala** (Scala 2.13, `sbt-assembly`); the control-plane service is **.NET**.
3. **Two artifacts:** the Scala Spark module is the *only* thing that touches Spark/Parquet/MinIO and
   writes results to Mongo; the .NET service reads the materialized `insights_*` collections via
   **database-service gRPC** and submits/monitors Spark jobs via the C# Kubernetes client.
4. **Annotations-first:** derive blunder / eval-swing / think-time from Lichess's embedded
   `%eval`/`%clk`; engine re-analysis is opt-in, throttled, off-peak — never the default.
5. **MinIO data lake**; materialize aggregates to a dedicated **`insights-db`**; client page consumes
   the REST/gRPC API. Elasticsearch surfacing is optional / non-priority. **Never touch the deprecated
   mono.**

## New repos / services / infrastructure to pre-create

Tell the user to create these in advance (out of scope to scaffold from nothing here):

- **`maichess-insights-service`** — new repo (.NET control plane + query API + the Scala `spark/`
  module co-located, or a sibling `maichess-insights-spark` repo).
- **`insights-db`** — a new **DatabaseService instance** (Mongo) for the `insights_*` collections,
  not a new repo (mirrors how `anticheat-db` was provisioned).
- **Infra (deploy-only, no repo):** **MinIO** (S3 data lake), the **Spark Operator** (Helm dep), and
  the custom **Spark image** (`ghcr.io/maichess/maichess-insights-spark`). Gated to staging first like
  search/anticheat.

## Tasks

| # | Task | Touches | Depends on |
|---|------|---------|-----------|
| 01 | ✅ [Contracts + `insights-db`](01-contracts-and-insights-db.md) — `insights.proto`, `rest/insights.md`, provision the DB | api-contracts, deploy | — (needs a contracts publish) |
| 02 | ✅ [MinIO + Spark Operator](02-minio-and-spark-operator.md) — data lake, operator, RBAC, node-pinning, Spark image + publish workflow | maichess-deploy | — |
| 03 | ✅ [Spark ingestion + PGN parser](03-spark-ingestion-and-parser.md) — Lichess `.zst` download, decompress-once, parse `%eval`/`%clk`, partitioned Parquet, manual upload | insights spark module | 02 |
| 04 | ✅ [Spark analysis jobs](04-spark-analysis-jobs.md) — opening / endgame / position / tricky / summary → Mongo `insights_*` | insights spark module | 01, 03 |
| 05 | ✅ [Control plane](05-insights-service-control-plane.md) — submit/track `SparkApplication` CRDs, monthly schedule, PGN upload endpoint | insights-service (.NET) | 01, 02 |
| 06 | ✅ [Query API](06-insights-query-api.md) — REST/gRPC reading `insights_*` via database-service; Redis L1 | insights-service (.NET), client contract | 01, 04, 05 |
| 07 | ✅ [Client Insights page](07-client-insights-page.md) — opening explorer, endgames, common/tricky positions, job submit + status | maichess-client | 06 |

**Dependency order:** `01 → 02 → 03 → 04 → 05 → 06 → 07`. `01` and `02` are independent of each other
and can run in parallel; `04` needs both the contract (`01`) and parsed Parquet (`03`); `05` needs the
contract (`01`) and the operator/RBAC (`02`); `06` needs jobs writing data (`04`) and the control plane
(`05`); `07` consumes `06`.

## Conventions for every task here

- **Contracts first, publish handoff.** Any change under `maichess-api-contracts/` stops to have the
  **user** tag/push a `vX.Y.Z`, then bumps `Maichess.PlatformProtos` in **all** consumers. A fresh
  agent's shell cannot restore a freshly published version — implement, document the blocker in
  `CONTRACT_NOTES.md`, and hand off (see [../../conventions.md](../../conventions.md)).
- **Persist via database-service.** The .NET service reads `insights_*` only through the generic
  DatabaseService gRPC CRUD contract against `insights-db` — never a direct Mongo driver. The Spark
  module writes via the Spark Mongo connector to the same store (its analytic write path is the one
  exception, documented in the ADR).
- **Annotations-first / no live-system pressure.** Jobs read Parquet, not live Kafka; default to
  filtered/sampled corpora; never call `engine-service` on the default path. Spark is node-pinned and
  resource-capped per [spark-and-minio](../../../knowledge/operations/spark-and-minio.md).
- **Testing.** .NET: 100% line/branch/method on non-excluded code, Reqnroll for domain logic, Stryker
  mirrored exclusions, `[ExcludeFromCodeCoverage]` only on I/O glue (Spark launcher, repositories, REST
  handlers, DTO records). Scala Spark: pure feature-extraction functions unit-tested with scalatest;
  transformations tested against a `local[*]` `SparkSession` over fixture PGNs.
- **Keep the KB current.** Update the three insights docs + `ROADMAP.md` (mark `🟡`/`✅`) as work ships.
