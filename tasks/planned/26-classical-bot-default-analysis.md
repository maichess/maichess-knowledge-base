# 26 — Make the knowledge ("classical") bot the default analysis engine + handle stale cache

> Read `conventions.md`,
> [analysis-service.md](../../knowledge/services/analysis-service.md), and
> [caching-and-read-models.md](../../knowledge/architecture/caching-and-read-models.md)
> first. Touches `maichess-analysis-service` (default-bot config + analysis cache).

## Goal (item 12)

1. Default game analysis to the **tier-5 knowledge ("classical") bot** instead of the
   current default engine.
2. Decide and implement what happens to analyses that were **cached with the old
   default** so users don't keep seeing stale/weaker lines.

## Part 1 — change the default

- The client reads `default_bot_id` from `GET /analysis/config` (see
  `app/analysis/[id]/page.tsx` → `AnalysisConfig`). Change the analysis-service config so
  the default resolves to the knowledge classical bot id.
- Verify the knowledge bot is registered/available to the analysis engine in every
  environment before flipping the default (otherwise sessions fail to start). Confirm via
  the bot registry / analysis-service bot list.

## Part 2 — "what about the wrongly cached games?"

Investigate the analysis cache key (the L1/read-model from tasks 09/12 and the
analysis-service's own result cache):

- **If the cache key already includes `bot_id` (+ line_count + fen/depth):** old entries
  are keyed under the *old* bot and simply won't be hit for the new default — no
  correctness problem, only cold cache for the new default. Document that and you're done;
  optionally pre-warm.
- **If the cache key does NOT include the engine identity** (i.e. cached as "the analysis
  for this position/game" regardless of bot): those entries are now **wrong** for the new
  default and must be invalidated. Two options:
  - **Add `bot_id` to the cache key** (preferred — future-proofs against any default
    change and lets per-bot results coexist), then old keys age out naturally.
  - Or **bump a cache version / flush** the affected analysis cache namespace on rollout.

Pick the keying fix (preferred) unless there's a strong reason not to; document the choice
in `caching-and-read-models.md` and the analysis-service `CLAUDE.md`.

## Testing

- Analysis-service: unit tests for default-bot resolution and the cache-key composition
  (asserting `bot_id` participates in the key). 100% on new non-excluded code; mirror
  Stryker exclusions.
- Manual: open a game analysis, confirm the classical bot is selected by default and lines
  reflect it (not a stale cached result).
