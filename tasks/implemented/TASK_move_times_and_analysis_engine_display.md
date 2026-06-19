# Task: Move clock times in PGN/analysis + analysis engine display

## Background

Two unrelated quality-of-life gaps in the analysis experience:

1. **No sense of time pressure.** When inspecting or importing a game, there is no
   way to see how long each side had on the clock at each move. The match read
   model carries authoritative per-move clocks in the `MoveApplied` event
   (`white_time_ms` / `black_time_ms`, see
   `services/maichess-match-manager-service/.../Kafka/MatchProjector.cs:282`), but
   **only the *current* clocks are persisted** on the durable match document
   (`white_time_ms` / `black_time_ms` scalars in `MatchRepository.cs:192`). There
   is no per-move clock history anywhere durable, so the analysis-service has
   nothing to surface. The generated PGN (`maichess-client/lib/utils/pgn.ts`
   `matchToPgn`, and the server-side PGN in `AnalysisGame.Pgn`) contains no clock
   annotations.

2. **The analysis engine is invisible.** The analysis view runs against a
   configurable bot (default `knowledge_classical`, see analysis-service
   `Analysis:DefaultBotId`), selectable under *Advanced settings*
   (`maichess-client/lib/components/analysis/AdvancedSettings.tsx`). Nothing in
   the UI tells the user which bot is currently analysing, nor whether it is the
   default. There is also no quick way to return to the default after
   experimenting.

This task delivers all three: move-time annotations end-to-end, an engine
indicator next to the line count, and a reset-to-default control.

---

## Part A — Move clock times in PGN and the analysis view

This part touches contracts and three services. The required contract additions
(`clock_history` on the match document, an optional `clock_history` array on the
analysis `GameDetail`, and PGN clock comments) are **pre-approved**: they are purely
additive (new optional array fields), so existing consumers are unaffected and you
may implement them directly. Update `maichess-api-contracts/rest/analysis.md` (and
any match read-model contract) as part of this task. Still record a one-line note in
the owning service's `CONTRACT_NOTES.md` describing what was added, for traceability.
No existing field may change type or meaning.

### A1. Persist per-move clock history (match-manager-service)

The `MatchProjector` and `MatchHistoryProjection` already see authoritative clocks
on every `MoveApplied`. Persist a parallel per-move clock array on the durable match
document alongside `moves` / `fen_history`:

- Add a field — proposed `clock_history` — a list of `{ white_time_ms, black_time_ms }`
  snapshots, one entry per applied move (index `i` = clocks **after** `moves[i]`).
  Ordering matches `moves` / `fen_history`.
- Populate it in the history projection (`MatchHistoryProjection` /
  `MatchProjector`) on each `MoveApplied`; seed empty on `MatchCreated`.
- Update `MatchRepository` serialization/deserialization (`MatchRepository.cs`
  ~192–240) to round-trip the new array.
- This is rebuildable from the event log, so no backfill is strictly required, but
  note in the task PR that pre-existing match documents will have an empty/absent
  `clock_history` and downstream code must treat it as "no clock data".

### A2. Surface clocks from the analysis-service

- **Match import** (`AnalysisGameService`, see analysis-service `CLAUDE.md` →
  "Match import"): when reading the match document via `Database.Get(collection="matches")`,
  read the new `clock_history` array if present and carry it onto the
  `AnalysisGame` domain record (add a `ClockHistory` field to
  `Domain/AnalysisGame.cs`).
- **PGN generation**: when building the PGN via `Moves.ConvertSequenceToSan(...)`,
  emit a `{[%clk H:MM:SS]}` comment after each move's SAN, using the remaining
  clock for the side that just moved (PGN-standard clock annotation). If
  `clock_history` is absent/empty, generate the PGN exactly as today (no comments)
  — never invent times.
- **PGN import** (`POST /games/from-pgn`): when an imported PGN already contains
  `{[%clk ...]}` (or `{[%emt ...]}`) comments, parse them into the same
  `ClockHistory` representation so imported games show times too. Imported PGNs
  without clock comments simply have none. Keep storing the original PGN text
  verbatim (current behaviour).
- **GameDetail response** (`Rest/GameDetailResponse.cs` + `AnalysisGameMapper.ToDetail`):
  add an optional `clock_history` array parallel to `moves` / `fens`. Omit or send
  empty when there is no clock data.

### A3. Client — PGN export and move-list display

- **Model**: add `clock_history?: { white_time_ms: number; black_time_ms: number }[]`
  to `AnalysisGameDetail` in `maichess-client/lib/models/analysis.ts`.
- **`matchToPgn`** (`lib/utils/pgn.ts`): when the source `Match` has clock data,
  emit `{[%clk H:MM:SS]}` after each move (reuse / extend `lib/utils/time.ts`
  `msToClockString`, but render in `H:MM:SS` PGN form). Without clock data, behave
  exactly as today.
- **Move list** (`lib/components/analysis/AnalysisMoveList.tsx`): show the remaining
  clock (and/or time spent on the move, derived from consecutive snapshots +
  increment) next to each SAN entry, in a muted `tabular-nums` style so the columns
  stay aligned. Degrade gracefully to today's layout when `clock_history` is
  absent. Keep the component a thin view — push any derivation into a hook/util per
  the client `CLAUDE.md` style rules.

**"Time spent" derivation** (optional, for the time-pressure cue): for move `i` by
a side, `spent = prevRemaining + increment_ms - remaining[i]`. Show remaining clock
as the primary value; time-spent is a nice-to-have if it reads cleanly.

---

## Part B — Show the current analysis engine + default indicator (client only)

`AnalysisConfig` already carries `default_bot_id` and `bots[]`
(`lib/models/analysis.ts`), and the session state exposes the active `botId`
(`useAnalysisSession` / `AnalysisClient` `state.botId`). No backend change needed.

- Near the **Lines** control — above or beside it, in `AnalysisPanel` header or just
  above `AdvancedSettings` — display the name of the bot the analysis is currently
  running against (resolve `state.botId` → `config.bots.find(...)?.name`, fall back
  to the id).
- Indicate whether it is the default: when `state.botId === config.default_bot_id`,
  show a subtle "Default" chip/badge; otherwise show a "Custom" (or no) badge so the
  user can tell they've overridden it. Match the existing badge styling used for the
  `BOT` / engine On-Off pills in `AnalysisClient.tsx`.
- This indicator should be visible whenever the engine is on (it lives in the
  engine-only sidebar area).

## Part C — Reset-to-default button (client only)

In `AdvancedSettings.tsx`:

- Add a **Reset to default** button (e.g. next to / below *Apply*). It calls the
  existing settings path with the config defaults — effectively
  `onApply(config.default_bot_id, config.default_line_count)` — and resets the local
  `localBotId` / `localLineCount` state to those defaults.
- `AdvancedSettings` currently only receives `bots`, `botId`, `lineCount`,
  `onApply`. Thread the defaults in (either pass `defaultBotId` / `defaultLineCount`
  props from `AnalysisClient`, which already has `config`, or pass `config`).
- Disable the reset button when the current applied settings already equal the
  defaults (mirror the existing `Apply` disabled logic).

---

## Constraints

- **Additive contracts only.** Part A's contract additions (`rest/analysis.md`, the
  match document shape, any proto touching the match read model) are pre-approved but
  must stay **additive and optional** — no existing field changes meaning or type.
  Note what was added in the owning service's `CONTRACT_NOTES.md` for traceability.
- **No invented data.** Every layer must degrade to "no clock info" cleanly for
  matches/PGNs that predate this feature or never had clocks. An empty/absent
  `clock_history` is a valid, common case.
- Do **not** touch `tournament-server/` or treat its OpenAPI as editable.
- Follow each service's style rules (analysis-service: sealed, records, warnings-as-
  errors; client: behaviour behind hooks, no duplication).

## Testing

- **match-manager-service** & **analysis-service**: 100% line/branch/method coverage
  on new non-excluded code (per each service's `CLAUDE.md`). Add cases for: clock
  history populated across multiple moves; empty/absent clock history; PGN clock
  comment emission; PGN clock-comment *parsing* on import (with and without
  `%clk`/`%emt`). Run `dotnet test -p:CollectCoverage=true` before completion.
  Consider a Stryker pass on the new clock-parsing/formatting logic — it's prone to
  off-by-one and rounding mutants that line coverage alone won't catch.
- **maichess-client**: no test framework configured; verify manually via
  `npm run dev` — import a finished native match and confirm clocks appear in the
  move list and in the exported PGN; confirm the engine name + default/custom badge
  render and update when the bot is changed; confirm *Reset to default* restores both
  bot and line count and becomes disabled. Run `npm run lint` and `npm run build`.

## Out of scope

- Live clock display during an in-progress game (this is the *analysis/review* view).
- Per-move evaluation/accuracy or clock-vs-eval graphs.
- Changing how clocks are computed or enforced during play (match-manager timing
  logic is unchanged; we only persist what it already computes).
- Backfilling `clock_history` onto historical match documents (acceptable to leave
  old games without times; mention it, don't build it unless asked).
