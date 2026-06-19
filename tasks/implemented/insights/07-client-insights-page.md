# 07 — Client Insights page

> Read [conventions.md](../../conventions.md) (the **Client conventions** section) and
> [domain/insights-statistics](../../../knowledge/domain/insights-statistics.md) first.
> `maichess-client` only. Depends on the query API (06). **Never touch the deprecated mono.**

## Goal

A new **Insights** page where users pick/launch a corpus and explore the results: openings, endgames,
common positions, and tricky positions, plus job submission + status.

## What already exists (reuse it)

- `maichess-client` (Next.js 16) conventions: server components by default, client behaviour in
  `lib/hooks/`, models in `lib/models/`, helpers in `lib/utils/`, `@/*` alias, REST proxy in
  `proxy.ts`/`routes.ts`. No test framework — verify with `npm run build` + `npm run lint` + manual
  click-through (conventions item: Client conventions).
- The existing **Tools** tab pattern (Bot arena / All games / Search live under `/tools/*`,
  `requireUser`) — add Insights there. Reuse the board renderer used by analysis/watch for the
  position/endgame/tricky FEN previews.

## Implementation

1. **`/tools/insights`** landing: list corpora (`ListCorpora`) with their filter/source + a **"Analyze
   a Lichess month"** form (month + rating/time/sample filter) and a **PGN upload** control, both
   calling task-05 endpoints; show job status (poll `GET /insights/jobs`, or subscribe to socket
   progress if task 01/05 added events).
2. **Opening explorer:** table/treemap of top openings by ECO/name with win/draw/loss bars and the
   month-over-month trend.
3. **Endgames:** table of material signatures by frequency + conversion tendency.
4. **Common positions & tricky positions:** ranked lists with a board preview per FEN; tricky view
   shows avg centipawn loss + think time (the [eval-loss ∩ think-time](../../../knowledge/domain/insights-statistics.md)
   intersection).
5. **Corpus summary** header: totals, rating distribution, draw rate, average length, termination mix,
   first-move popularity.
6. Data fetching in `lib/hooks/`, response models in `lib/models/`; route the new endpoints through the
   client REST proxy.

## Contract changes

None (consumes task 01's REST surface).

## Tests (mandatory)

- No client test framework: `npm run build` + `npm run lint` clean, plus a manual click-through of each
  view against a populated staging corpus.

## Verify

- On staging: launch a small Lichess-month analysis and a PGN upload from the page, watch the job reach
  `succeeded`, then browse openings / endgames / common & tricky positions with board previews and the
  summary header — all reflecting the corpus.

## Knowledge base

- Mark task 07 `✅` in [ROADMAP.md](../../ROADMAP.md) when the page ships (and move the whole program's
  specs to `implemented/insights/` once the end-to-end flow is verified live, per conventions item 8).
