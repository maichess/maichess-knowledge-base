# 29 — Strongest-bot search/eval hardening + variant-aware multi-PV analysis

> `maichess-engine-service` only. Follow-up to
> [26](26-classical-bot-default-analysis.md) (which made `knowledge_classical` the default
> analysis bot). No contract change, no version bump. Shipped 2026-06-14; engine tests
> 217 → 224, statement + branch coverage stay 100%.

## Context

Task 26 surfaced that multi-PV `AnalyzePosition` ran the tier-1 `Search` for *every* non-Basic
bot, so the analysis bot's identity didn't affect the lines. After fixing that (multi-PV now
dispatches on variant), the user asked to also harden the strongest bot (`knowledge_classical`
→ `SearchV5` over `SearchV4`). Improvements #1–#7 and #9 were implemented inline; #8 (tablebase
infra) was deferred to [planned/30](../planned/30-strongest-bot-tablebase-hardening.md).

## Delivered

**Variant-aware multi-PV analysis.** `EngineServiceLive.newMultiPvSearch` dispatches on
`EngineVariant`: `Base`→`Search`, `EnhancedSearch`→`SearchV2`, `EnhancedOrdering`→`SearchV3`,
`EnhancedEval`/`Knowledge`→`SearchV4`. The four tiers share a new `chess/MultiPvSearch` trait
(`bestMoveAtDepth` + `extractPv`). Knowledge has no multi-line analogue for its book/tablebase
oracles, so it analyses with `SearchV4`.

**Strongest-bot search/eval (all in `SearchV4` / `EvalV2` / `OpeningBook`, so they reach tier 4
play, tier-5 play via `SearchV5`'s inner `SearchV4`, and tier-4/5 analysis):**

1. **In-search draw detection** — repetition along the search path scores `0` (bounded by the
   half-move clock); 50-move rule (`halfMoveClock >= 100`, suppressed in check so it can't
   override mate). Path-dependent → never stored in the TT. Catches only search-created
   repetitions (the engine gets a bare FEN, no prior game history) — enough to stop repeating in
   a won position or to claim a perpetual when worse.
2. **Ply-adjusted mate scores** across the TT (`toTT`/`fromTT`) — mate distance stays correct
   regardless of store-ply vs probe-ply.
3. **Bishop-pair bonus** in `EvalV2` (+30, white-positive then signed to stm).
4. **Knight/bishop mobility now masks own-occupied squares** (`& ~own`), matching rook/queen —
   previously knight/bishop counted own pieces as mobility.
5. **New eval terms** in `EvalV2`: **knight/bishop outposts** (relative ranks 4–6, pawn-defended
   and unreachable by an enemy pawn via a new `pawnSpanMask`), **pawn threats** (own non-pawn
   piece attacked by an enemy pawn, value-scaled penalty), and a **tempo** bonus (+10, constant,
   side-to-move). Tempo makes the eval antisymmetric *modulo a constant*: a mirror-symmetric
   position now scores `Tempo`, and `evaluate(wtm) + evaluate(btm) == 2·Tempo` (the three EvalV2
   symmetry tests were updated accordingly).
6. **Depth-preferred TT replacement** — a deeper entry for a *different* position survives an
   index collision; same-position / empty slots always refresh. (Fresh TT per search, so no
   generational aging.)
7. **Analysis mode** — `bestMoveAtDepth` sets `analysisMode`, disabling NMP + LMR so every
   multi-PV line's score is a faithful evaluation, not a fast best-move lower bound. `bestMove`
   (play) keeps NMP/LMR on.
9. **Opening book → highest-weight move** (`OpeningBook.maxWeightIndex`), deterministic max
   strength, replacing the `ThreadLocalRandom` weighted sampling. Affects all Knowledge bots.

## Notes / decisions

- Search changes scoped to **`SearchV4`** (the strongest bot's search; `SearchV5` wraps it).
  Lower tiers (`Search`/`SearchV2`/`SearchV3`) are deliberately left at their existing strength
  per the "copy, don't modify lower tiers" convention. So `search_*`/`ordering_*` analysis does
  not get analysis-mode/draw-detection — acceptable, they aren't the strongest bot.
- `EvalV2` is shared by tiers 4 and 5 only, so the eval changes scope to the strong bots
  naturally.
- The book change is deterministic for **all** Knowledge bots (book is shared); this removes
  opening variety in exchange for always playing the top-weighted theory move (user chose max
  strength).

## Tests

- `SearchV4Spec`: 50-move draw at clock limit + winning contrast at clock 0 + forced-perpetual
  repetition draw (verified FEN where Black's a-file Q/R genuinely cannot interpose) + analysis-
  mode mate-in-1.
- `EvalV2Spec`: bishop-pair mirror; knight-mobility isolation; outpost isolation (c4 vs a4
  defender, all other terms equal); pawn-threat isolation (c6 vs f6, only the threat differs);
  the three symmetry tests updated for tempo.
- `OpeningBookSpec`: multi-entry buffer → highest-weight move chosen deterministically.
- `EngineServiceSpec`: per-variant `analyzePosition` dispatch (one bot per tier).
- `sbt clean coverage test coverageReport` → 226 passed, stmt 100% / branch 100%.

## Follow-ups

- [planned/30](../planned/30-strongest-bot-tablebase-hardening.md) — tablebase source/cache/
  analysis (engine improvement #8), details left as open questions.
- Lower-tier analysis-mode/draw-detection parity is possible but intentionally not done.
