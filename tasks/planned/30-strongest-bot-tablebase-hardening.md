# 29 — Strongest-bot endgame tablebase hardening (local source + caching + analysis WDL)

> Read `conventions.md` and the engine `CLAUDE.md` (tier 5 / `SearchV5`) first.
> `maichess-engine-service` only. This is engine improvement **#8** deferred from the
> 2026-06-14 strongest-bot pass (#1–#7 and #9 were implemented inline; see
> [implemented/26](../implemented/26-classical-bot-default-analysis.md) follow-up and the
> engine commits). **Details are deliberately left open** — decide them at implementation
> time with current infra in view.

## Problem

The tier-5 Knowledge bot (`knowledge_classical` and siblings) probes the endgame tablebase
via a **per-move HTTP call to the public Lichess endpoint**
(`service/clients/TablebaseClientLive`, `https://tablebase.lichess.ovh/standard`, 2 s
timeout, silent fallback to search on any failure). Consequences:

- **Latency:** every position with ≤7 pieces pays up to a 2 s network round-trip, eating into
  the move-time budget (worst for `Fixed(5000)` classical and the proportional/aggressive
  variants that already spend more clock).
- **Reliability / strength:** offline, rate-limited, or slow network → the probe returns
  `None` and the bot plays the (weaker) search move instead of provably-perfect endgame play.
  This is invisible (no error surfaced).
- **Not on the analysis path at all:** multi-PV `AnalyzePosition` never probes the tablebase,
  so low-piece analysis lines are pure search even for the Knowledge bot. (Multi-PV analysis
  now dispatches on variant — Knowledge analyses with `SearchV4` — but has no tablebase WDL/DTZ.)

## Goal

Make low-piece play (and optionally analysis) for the strongest bot fast and reliable, without
depending on a flaky per-move external call.

## Open questions (decide at implementation time)

1. **Tablebase source.** Bundle local **Syzygy** `.rtbw`/`.rtbz` files (3–4–5 men ≈ a few hundred
   MB; 6-men is ~150 GB; 7-men impractical to bundle) and probe in-process? Or stand up a
   **self-hosted** tablebase HTTP service (e.g. lila-tablebase) inside the cluster and point
   `TablebaseClientLive` at it? Or keep Lichess but add a cache/circuit-breaker? Trade-off:
   image size / ops vs. piece-count coverage vs. latency. What piece count do we actually need
   (current cap is `MaxPieces = 7`)?
2. **Caching.** Add a process-local and/or Redis cache of `fen → TablebaseResult` (positions
   recur within a game and across arena series). Keying, size, eviction, and whether the
   existing shared Redis is appropriate are open.
3. **Timeout / fallback policy.** Current 2 s blanket timeout — should it scale with the move
   budget? Should a tablebase miss be logged/metered (it is currently silent) so we can see how
   often the bot is denied perfect endgame play?
4. **Analysis integration (optional, depends on #1).** Should multi-PV `AnalyzePosition` for the
   Knowledge variant fold tablebase WDL/DTZ into the low-piece lines (e.g. surface a "tablebase:
   win in N" line)? This needs an effectful probe inside the currently-pure `searchMultiPv`
   path — non-trivial; only worth it with a fast local source.
5. **Scope.** Tier-5 only, or expose the local source to any future endgame-aware tier?

## Non-goals

- No change to the search/eval tiers themselves (that work shipped 2026-06-14).
- No contract change expected (engine-internal); confirm when the source decision is made.

## Testing

- Keep `TablebaseClient.parseResponse` unit tests; add tests for whatever new source/cache seam
  is introduced (mirror the `noop`/mocked-client pattern — the live network path stays
  `[ExcludeFromCodeCoverage]` / scoverage-excluded).
- Bench move latency on a ≤7-piece position before/after to confirm the latency win.
