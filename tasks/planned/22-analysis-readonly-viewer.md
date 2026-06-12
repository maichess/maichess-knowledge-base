# 22 — Analysis view: read-only (no-engine) mode + stop surfacing gRPC "Cancelled"

> Read `conventions.md` and `maichess-client/CLAUDE.md` first. Primarily a
> `maichess-client` change (`AnalysisClient` + `useAnalysisSession`); may touch the
> analysis-service session API only if a server-side "no-analysis" session is preferred.
> Bundles three related reports: **disable-analysis toggle (5)**, **clickable
> search/all-games rows → read-only analysis (8)**, and **the recurring "Cancelled"
> error (9)**.

## Goal

1. Add an **"Engine analysis" on/off toggle** to the game analysis view so it can be
   used as a plain past-match viewer (board + move list + navigation, no engine).
2. Make search / all-games result rows open the analysis view with **analysis
   deactivated** by default (deep link, e.g. `?analysis=off`).
3. Eliminate the `Status(StatusCode="Cancelled", Detail="Call canceled by the client.")`
   error that regularly appears in the lines window.

## Background — current architecture

`useAnalysisSession` (in `lib/hooks/`) creates a **server session** on mount and
auto-starts analysis. Navigation, whatif, and "current FEN" are all computed
server-side via the session API (`/api/sessions/*`). The reducer's `NAVIGATED` case
hard-codes `analysisRunning: true` with the comment *"navigation always restarts
analysis server-side"* — so every move-list click cancels the in-flight engine stream
and starts a new one. **That client-cancellation is what surfaces as the "Cancelled"
gRPC status** in `AnalysisPanel` (via `onError`).

Key fact for read-only mode: `AnalysisGameDetail` already carries a precomputed
`fens: string[]` and `starting_fen`, so navigation can be done **purely client-side
with no session at all**. Index semantics (from `AnalysisMoveList`): index `0` =
starting position, index `k` = after `k` moves.

## Implementation

### A. Analysis-off mode (items 5 + 8)

- Thread an `analysisEnabled` flag into `AnalysisClient` from a search param:
  `app/analysis/[id]/page.tsx` reads `searchParams` (a Promise in Next 16) and passes
  `analysisEnabled={searchParams.analysis !== 'off'}`. Add an in-view toggle that flips
  it at runtime.
- Extend `useAnalysisSession(game, config, enabled)`:
  - When `enabled` is `false`: **do not create a server session**; navigate
    client-side using a `fenAtIndex(game, index)` helper (handle both
    `fens.length === moves.length` and `=== moves.length + 1`); keep `currentLines`
    empty; disable whatif (board already disables when there's no session).
  - When toggled back on: create the session and start analysis.
  - Add a `NAVIGATED_LOCAL` action that sets index/fen **without** setting
    `analysisRunning: true`.
- `AnalysisPanel`: render a muted "Engine analysis off" state when disabled.
- Gate the engine-arrow overlay and the navigation buttons on a `ready` value that is
  `true` in disabled mode (today they gate on `hasSession`).
- Wire result rows: `AllGamesPanel` and `SearchPanel` rows should open
  `ROUTES.analysisGame(id) + '?analysis=off'`. They currently reference matches, so
  reuse the existing **import-from-match** flow (`POST /api/games/from-match/:matchId`
  → analysis game id → push with `?analysis=off`), mirroring `UserMatchList` and the
  new `AnalyseGameButton`. (See item-8 row also in task 25's all-games work.)

### B. Stop surfacing "Cancelled" (item 9)

Even with analysis on, a client-initiated cancel is **not** an error to show the user.
Two layers, do both:
- In the analysis socket/error path (`useAnalysisSocket` → `onError`), **ignore**
  statuses whose code is `Cancelled`/message contains `OperationCanceledException` /
  "Call canceled by the client" — these are expected when navigation restarts analysis.
- Debounce rapid navigation so we don't spawn-then-immediately-cancel a stream on every
  arrow press.
- Read-only mode (A) removes the cause entirely when the user just wants to review.

## Testing

No client test framework — verify with `npm run build`, `npm run lint`, and a manual
click-through: toggle analysis off/on; rapid-navigate (confirm no "Cancelled" text);
open a row from all-games/search and confirm it lands in the read-only viewer.
