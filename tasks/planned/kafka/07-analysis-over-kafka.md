# Kafka 07 — Analysis over Kafka

> Read the [README](README.md), the analysis hard-problem note in
> [event-driven-architecture](../../../knowledge/architecture/event-driven-architecture.md), and
> [analysis-service](../../../knowledge/services/analysis-service.md).
> **Depends on:** `01` (proto), `04` (engine streaming pattern). Independent of the match loop.

## Goal

Move analysis session control onto Kafka: `StartAnalysis`/`StopAnalysis` on `analysis.commands.v1`;
the engine streams `AnalysisDepthCompleted` to `analysis.events.v1`; cancellation is by `sessionId`.
This replaces the synchronous gRPC analysis stream (the ADR accepts losing native gRPC backpressure).

## Current state
Analysis logic still lives in **maichess-mono** (the standalone analysis-service is contracts-only).
`analysis.commands.v1`/`analysis.events.v1` topics exist but are unused.

## Implementation
- **Commands:** producer emits `StartAnalysis{sessionId, fen/pgn, depth, …}` and
  `StopAnalysis{sessionId}` to `analysis.commands.v1` (keyed by `sessionId`).
- **Engine:** consume `analysis.commands.v1`; for `StartAnalysis`, run the iterative-deepening search
  and stream `AnalysisDepthCompleted{sessionId, depth, score, pv, …}` to `analysis.events.v1` after
  each depth; on `StopAnalysis` (or a newer command for the same `sessionId`) cancel the running
  search. Reuse the engine's existing analysis/search code; add the consumer/producer behind the
  ScalaPB serde from `01`. Dedupe/cancel keyed on `sessionId`.
- **Consumer/UI side:** whoever currently consumes the gRPC analysis stream (maichess-mono / client
  path) subscribes to `analysis.events.v1` filtered by `sessionId` and forwards depth updates (via
  the existing socket/SSE channel) to the client.
- Decide where analysis orchestration lives now (keep in maichess-mono vs. begin extracting the
  analysis-service) — follow `analysis-service` knowledge doc; do not expand scope beyond wiring the
  Kafka control/stream here.

## Contract changes
None beyond `01` (uses the `analysis.*` protos). Publish handoff if anything under contracts changes.

## Tests
- Command round-trip; `StartAnalysis` → a sequence of increasing-depth `AnalysisDepthCompleted`;
  `StopAnalysis`/superseding command cancels (no further events for that `sessionId`).
- Serde round-trip; stream glue excluded, the session/cancel logic covered.
- Scala: `sbt test`/`sbt stryker`; any C#/mono piece to its coverage bar.

## Verify
- Start an analysis from the client → depth updates stream in and stop on cancel; two sessions don't
  cross-talk (keyed by `sessionId`).

## Docs to update
- `analysis-service` — document the Kafka control/stream model and where orchestration lives.
