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

## Implementation notes (done 2026-06-14)

**Part 1 — default flipped.** `Analysis__DefaultBotId` changed from `blitz` to
`knowledge_classical` (tier-5 Knowledge, elo 2600) in
`maichess-deploy/helm/maichess/values.yaml`. No env-specific values file
(`values-prod`/`values-staging`/`values-vps`) overrides this key, so the single change flips
the default in every environment. Local `services/maichess-analysis-service/.env_prod`
placeholder set to the same value. The client reads `default_bot_id` from
`GET /analysis/config` dynamically — **no client change**.

**Availability verified.** `knowledge_classical` is registered in the engine's `BotRegistry`
and the `AnalyzePosition` (multi-PV) stream rejects **only** `Basic`-variant bots
(`EngineServiceLive`, line ~28), so the Knowledge bot starts analysis sessions fine. The
SearchV5 tablebase dependency is *not* on the analysis path (see nuance below), so there is no
new external runtime dependency for analysis.

**Part 2 — stale cache: no keying change needed.** The cache key **already includes `bot_id`**
at both layers — Mongo L2 filter `{fen, bot_id}` (`AnalysisResultRepository`) and Redis L1 key
`analysis:{botId}:{fen}` (`RedisAnalysisResultCache`). This is the task's "no correctness
problem, only a cold cache" branch: old `blitz` entries are keyed under `blitz` and are never
hit for the new default. The startup bot-change scrape is the backstop — on the flip deploy
`stored_bot_id` (`blitz`) ≠ `DefaultBotId`, so `analysis_results` is purged from Mongo and the
L1 cleared before serving. No pre-warm. Documented in `caching-and-read-models.md`,
`analysis-service.md`, and the analysis-service `CLAUDE.md`.

**Engine nuance discovered (documented, out of scope to fix).** The engine's multi-PV
`AnalyzePosition` runs the **tier-1 `Search` for every non-`Basic` bot** (`searchMultiPv` in
`EngineServiceLive`), so the analysis *lines* are identical regardless of which bot a session
selects — `bot_id` only gates Basic rejection and caching. Flipping the default is therefore a
UX/labelling change today, not a change in analysis output. Keying the cache by `bot_id` is
forward-compatible: if per-bot analysis depth ever lands, no cache migration is required. This
is recorded in `caching-and-read-models.md`.

**Tests.** Existing `CachingAnalysisResultRepositoryTests` already assert `bot_id` participates
in the key and default-bot resolution. Added `DefaultAnalysisBotCachingTests` to pin the new
`knowledge_classical` default: the default bot is the one cached in L1 (keyed by its id), the
former default (`blitz`) now bypasses L1 as a non-default bot, and only the default bot's depth
writes append to L1. `dotnet test -p:CollectCoverage=true` → 121 passed, 100% line/branch/method.

**No contract change / no version bump.** `default_bot_id` is a config value, not a contract
constant (the `rest/analysis.md` JSON is only an example), so no proto/REST change and no
`platform-protos` version bump was required.
