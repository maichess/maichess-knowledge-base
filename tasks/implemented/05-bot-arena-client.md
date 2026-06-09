# 05 — Bot Arena client (Dev section UI)

**Goal:** Build the client Dev section that drives the arena service from `04`:
forms to create each setup type, live games watchable via the existing Watch flow,
and a Results tab that renders finished collections with a **type-specific view**.

> Read `conventions.md` first. Depends on `01` (the `dev_mode` gate +
> `/dev` shell) and `04` (the arena service + REST contract). Follow the Next.js 16
> client conventions in `maichess-client/CLAUDE.md`.

## Background to read first

- `01` left `app/dev/page.tsx` as a guarded "Dev tools" shell and a "Dev" nav entry
  shown when `user.dev_mode` is true. Build the real Dev pages under `app/dev/`.
- Arena REST: `maichess-api-contracts/rest/bot-arena.md` (from `04`).
- Reuse: the existing **Watch** feature (`app/watch/page.tsx`, `app/watch/[id]`,
  `lib/components/WatchClient.tsx`) — arena games are ordinary bot-vs-bot matches, so
  link live games straight into the existing match/watch viewer; do **not** build a
  second board.
- Reuse patterns: `lib/hooks/useBots.ts` (bot picker), `useTimeFormats.ts`,
  `lib/components/ui/*`, the proxy pattern in `app/api/**/route.ts` +
  `lib/utils/proxy.ts`.

## Structure

- **Proxy routes** under `app/api/dev/arena/...` forwarding to the arena service
  (bearer auth via `getBearerToken`, same shape as `app/api/matches/bot-vs-bot/route.ts`):
  create collection (per type), list collections, get collection, get/set the global
  concurrency limit.
- **Models** in `lib/models/arena.ts`: setup types, config shapes, collection +
  typed result views (bracket / matrix / single series).
- **Hooks** in `lib/hooks/`: `useArenaCollections` (list + poll running ones),
  `useArenaCollection` (one collection, polled while running), `useArenaConfig`
  (global concurrency get/set), `useCreateArenaSetup`.

## Pages (all gated by `dev_mode`)

Use the `01` guard helper on every `app/dev/**` page.

1. **`app/dev/page.tsx`** — Dev landing: links to "Spawn setups", "Results", and the
   "All games" browser (delivered by `07`), plus the **global concurrency limit**
   control (any user can edit it; show that it is global/shared).
2. **Spawn** (`app/dev/arena/new` or tabs on the landing):
   - **Tournament** form: multi-select bots, editable **FEN list** (with labels),
     time format. Explain best-of-3 / both-colors semantics inline.
   - **Matrix** form: multi-select bots, FEN list, games-per-FEN, time format
     (alternating colors noted as automatic).
   - **Single** form: white bot, black bot, FEN list (default standard), games-per-FEN,
     **keep-switching-colors** toggle, time format.
   - On submit → create via proxy → route to the collection detail page.
3. **Results / collections** (`app/dev/arena` list + `app/dev/arena/[id]` detail):
   - List finished (and running) collections with name, type, status, counts, date.
   - Detail renders **by `SetupType`**:
     - **Tournament → progression tree**: rounds as columns, pairings as nodes showing
       both bots, the best-of-3 game results, and the advancing winner. Each game links
       to the Watch/match viewer.
     - **Matrix → grid**: bots on both axes, each cell the aggregate score (and a
       drill-down to that pairing's games).
     - **Single → labelled list**: one row per game with a **telling name** — clear FEN
       label + who was White / who was Black + result. Each row links to the viewer.
   - Running collections show live progress (poll the arena `getCollection`); in-flight
     games link to the live Watch view.

## Verification

- `cd maichess-client && npm run lint && npm run build`.
- Manual end-to-end (with `04` + services running): enable Dev mode (`01`) → "Dev"
  appears in nav → create each setup type → games show in Watch and the concurrency
  cap is respected → finished collections render the correct typed view and game links
  open the existing match viewer.

## Knowledge base

Note in `maichess-knowledge-base/` that the Dev section is `dev_mode`-gated, reuses
the Watch viewer for live arena games, and renders results per setup type
(tree / matrix / labelled list).
