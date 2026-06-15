# 05 — insights-service control plane

> Read [conventions.md](../../conventions.md),
> [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md), and
> [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md) first.
> `maichess-insights-service` (.NET net10.0). Depends on the contract (01) and the Operator/RBAC/MinIO
> infra (02).

## Goal

Stand up the .NET control plane: accept ingestion/analysis requests and manual PGN uploads, launch and
track Spark jobs as `SparkApplication` CRDs, and run the monthly Lichess pull on a schedule.

## What already exists (reuse it)

- Service layout + DI + OTel + auth + Redis seam patterns from `maichess-analysis-service`
  (`Domain/`, `Data/`, `Services/`, `Rest/`, `Grpc/`, `Kafka/`, `Program.cs`, `*.Tests/`).
- The RBAC `ServiceAccount` + `SparkApplication`/`ScheduledSparkApplication` CRDs from task 02.
- DatabaseService gRPC access for the `insights_jobs` catalog (`insights-db`) — same `Database.DatabaseClient`
  + `Struct` pattern user-service uses; **no direct Mongo driver** (conventions item 6).
- MinIO SDK for storing uploaded PGNs into `insights-raw`.

## Implementation

1. **`Services/JobService`** (fully covered domain logic): validate a request, choose which job(s) to
   run, build the corpus id + filter, and decide the `SparkApplication` spec parameters. Pure
   decision/validation logic — testable without k8s.
2. **`Data/SparkJobLauncher`** (`[ExcludeFromCodeCoverage]`, infra adapter like `MatchRepository`):
   create `SparkApplication` resources via the C# Kubernetes client; watch/poll their status and write
   it into `insights_jobs`. Behind an `ISparkJobLauncher` seam so `JobService` is tested with a fake.
3. **Manual PGN upload** (`Rest/` handler → service): accept a multipart multi-game PGN, store it to
   MinIO `insights-raw`, create an ingestion `SparkApplication` pointed at the object key, return the
   job/corpus id.
4. **`SubmitIngestion` / `SubmitAnalysis` / `GetJob` / `ListJobs`** REST + gRPC, mapping to
   `JobService` + the `insights_jobs` catalog.
5. **Monthly schedule:** a `ScheduledSparkApplication` (Helm) for the latest Lichess standard month,
   running **off-peak**, with the default filter/sample; the service surfaces its runs in `ListJobs`.
6. *(Optional)* if task 01 added `insights_events.proto`, emit job-lifecycle events to
   `socket.outbound.v1` for live progress.

## Contract changes

None new (implements task 01's contract). If a field is missing, follow the publish handoff in
[conventions.md](../../conventions.md) and record any blocker in `CONTRACT_NOTES.md`.

## Tests (mandatory)

- `dotnet test -p:CollectCoverage=true` — **100%** line/branch/method on non-excluded code. Reqnroll
  scenarios for job-dispatch (request → correct `SparkApplication` spec via a fake launcher), upload
  routing, and status mapping. Stryker.NET wired (`.config/dotnet-tools.json` + `stryker-config.json`)
  mirroring the exclusions.
- Excluded: `SparkJobLauncher`, repositories, REST handler classes, MinIO client glue, `Program.cs`,
  generated/logging partials, REST DTO records.

## Verify

- On staging: `POST /insights/ingestions` for a small Lichess slice and `POST /insights/uploads` for a
  multi-game PGN each create a `SparkApplication` that runs on `maichess-mega`; `GET /insights/jobs`
  reflects status transitions; the monthly schedule fires once at its configured hour.

## Knowledge base

- Update [insights-and-spark](../../../knowledge/architecture/insights-and-spark.md) if the
  launch/seam design diverges. Mark task 05 `🟡` in [ROADMAP.md](../../ROADMAP.md).
</content>
