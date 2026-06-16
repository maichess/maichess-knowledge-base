# 01 — Insights contracts + `insights-db`

> Read [conventions.md](../../conventions.md),
> [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md), and
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md) first.
> Touches `maichess-api-contracts/` only (+ a deploy note to provision `insights-db`). **Contracts are
> the source of truth — author this before any service code.**

## Goal

Define the insights service contract (gRPC + REST) and provision the dedicated `insights-db`
DatabaseService instance, so subsequent tasks implement against a published contract.

## What already exists (reuse it)

- Proto style + layout: `maichess-api-contracts/protos/<service>/v1/`, events under
  `protos/events/v1/`, REST specs under `rest/<service>.md`. Follow
  `maichess-api-contracts/CLAUDE.md` (snake_case fields, PascalCase messages/services,
  SCREAMING_SNAKE_CASE enums, one service per file, `v1/`).
- The publish handoff + multi-consumer version bump (conventions item 2) — current published version is
  `0.13.0`; the next tag is the new baseline for **all** consumers.
- `insights-db` follows the `anticheat-db` precedent: a new DatabaseService Mongo instance, not a repo.

## Implementation

1. **`protos/insights-service/v1/insights.proto`** — service `Insights` with:
   - Job control: `SubmitIngestion` (source descriptor: lichess month / upload key / future source +
     filter: rating band, time control, date slice, sample rate), `SubmitAnalysis` (corpus id + which
     jobs), `GetJob`, `ListJobs`.
   - Queries (all keyed by `corpus_id`, paged): `GetTopOpenings`, `GetCommonEndgames`,
     `GetCommonPositions`, `GetTrickyPositions`, `GetCorpusSummary`, `ListCorpora`.
   - Messages mirror the metric/schema definitions in
     [insights-statistics](../../../knowledge/domain/insights-statistics.md) (opening row, endgame
     signature row, position row, tricky row, summary, job record).
2. **`rest/insights.md`** — REST surface: `POST /insights/ingestions`, `POST /insights/uploads`
   (multipart multi-game PGN), `POST /insights/analyses`, `GET /insights/jobs[/{id}]`,
   `GET /insights/corpora`, and the query endpoints (`/insights/corpora/{id}/openings|endgames|positions|tricky|summary`).
3. *(Optional, only if task 05 wants live job progress.)* `protos/events/v1/insights_events.proto` —
   job lifecycle events for `socket.outbound.v1`. Defer unless needed.
4. **Provision `insights-db`** — add the deploy note / values entry for a new DatabaseService Mongo
   instance (the actual Helm wiring lands in task 02 / 05). Collections per
   [insights-statistics](../../../knowledge/domain/insights-statistics.md): `insights_openings`,
   `insights_endgames`, `insights_positions`, `insights_tricky`, `insights_summary`, `insights_jobs`.

## Contract changes

- New `insights.proto` + `rest/insights.md` (additive — `buf breaking` is a no-op for a new service).
- **Stop and instruct the user** to commit, tag (`vX.Y.Z`, the next after `0.13.0`), and push the
  contracts repo so `Maichess.PlatformProtos` (C#/Scala/TS) builds. Then bump the pinned version in
  **all** consuming services to the newly published version (reconcile drift, not just insights).
- If a consumer can't be verified because the package isn't published yet, record the blocker in that
  service's `CONTRACT_NOTES.md` and stop.

## Tests (mandatory)

- No runtime code here; verify `buf lint` + `buf generate` succeed and the generated package builds in
  each language. Coverage gates apply to the services that consume this contract (tasks 04–07).

## Verify

- `buf lint` / `buf breaking` clean; generated C#/Scala/TS types present after the tagged publish.

## Knowledge base

- This task's design already lives in
  [insights-and-spark](../../../knowledge/architecture/insights-and-spark.md) and
  [insights-statistics](../../../knowledge/domain/insights-statistics.md); update them if the contract
  shape diverges. Mark task 01 `🟡` in [ROADMAP.md](../../ROADMAP.md) when the contract is published.
</content>
