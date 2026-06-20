# Task: Close mutation testing gaps in maichess-analysis-service

## Background

A first Stryker.NET run on `services/maichess-analysis-service` reports a
mutation score of ~47%. After excluding the untested infrastructure files
(see "Excluded files" below), 96 surviving mutants remain — all in
`Services/AnalysisGameService.cs`. The current config lives at
`services/maichess-analysis-service/MaichessAnalysisService.Tests/stryker-config.json`.

Two distinct problems:

1. **Entire `AnalysisSessionService` has no tests.** Stryker reports every
   line in `Services/AnalysisSessionService.cs` as `NoCoverage`, plus the
   `Domain/AnalysisSession.cs` entity helpers (`GetCurrentFen`, `GetBaseFen`).
   The service `CLAUDE.md` requires 100% line/branch/method coverage and
   names this class explicitly, so this is a real violation of the project's
   own testing policy.
2. **`AnalysisGameService` has wide test-quality gaps.** Many strings (DB
   field names, PGN format keys), pagination math, list ordering, and
   field-extraction code are not exercised in a way that distinguishes the
   correct value from a mutated one.

## Excluded files (intentional — do not touch)

The stryker-config.json now excludes:
- `Domain/AnalysisSession.cs` — until session tests exist (see below)
- `Domain/*Exception.cs` — exception message strings carry no behaviour
- `Services/AnalysisSessionService.cs` — until session tests exist
- `Services/AnalysisConfig.cs` — record with config defaults
- `Data/Analysis*Repository.cs` — already `[ExcludeFromCodeCoverage]` (live gRPC)
- `Rest/**/*Endpoints.cs` — REST adapters

## Goal

Drive the mutation score on the non-excluded code to ≥ 95%, then re-enable
mutation on `AnalysisSession.cs` / `AnalysisSessionService.cs` and do the
same there.

## Phase 1 — Close `AnalysisGameService` gaps

Run `cd MaichessAnalysisService.Tests && dotnet stryker` and open
`StrykerOutput/<timestamp>/reports/mutation-report.html`. The categories
below cover the surviving mutants seen on 2026-05-27. Fix them by tightening
tests, not by adding ignores (unless the mutation is genuinely equivalent).

### 1a. Match-PGN round-trip

Lines 265–347 — `BuildMatchTags`, `BuildMatchPgn`, and the surrounding
helpers — build a full PGN string from a stored match record. Surviving
mutants include:
- PGN tag keys: `"Event"`, `"Site"`, `"Date"`, `"Result"` (L308–311) → ""
- White/black name extraction keys: `"user_id"`, `"bot_id"` (L321–324) → ""
- Move-number arithmetic (L338 `i % 2`, L340 `(i / 2) + 1`) → various
- StringBuilder append statements removed

**Fix:** in `Features/ImportFromMatch.feature` (or a new feature file), add
a scenario that imports a match with a known move sequence and asserts the
exact generated PGN string (multi-line). One end-to-end string assertion
kills most of these mutants at once.

### 1b. `BuildPlayerInfo` branching

Lines 298–303. Three branches: `user_id` present, `bot_id` present, or
empty dictionary. Surviving mutants include the `userId.Length > 0` /
`botId.Length > 0` equality flips.

**Fix:** add an internal-visible test (the project already exposes internals
to the test assembly) that covers all three branches plus the
empty-but-non-null edge cases (`""`, `"   "`, `null`).

### 1c. `UserMatchSummary` field extraction

Lines 351–374 — `BuildUserMatchSummary`. Surviving mutants include the DB
field name strings (`"id"`, `"status"`, `"white_user_id"`, `"black_user_id"`,
`"white_bot_id"`, `"black_bot_id"`, `"moves"`) and the `?? string.Empty`
fallback for `status`.

**Fix:** the existing `Features/ListUserMatches.feature` should be extended
to (a) construct a Mongo-style `Struct` with each field set to a distinct
sentinel value, and (b) assert that every field of the resulting
`UserMatchSummary` matches the corresponding sentinel. This is the same
class of bug as the `password_hash` field in user-service (already fixed
there) — wrong field name = silent data loss.

### 1d. `ReadTimeFormat` and the legacy fallback

Lines 376–395 — reads `time_format_*` fields, or falls back to a `time_control`
string mapped via a switch. The mutants here are mostly `NoCoverage` —
meaning no test exercises a record with `time_format_id` set, and no test
exercises the legacy path. The L388 `?? "blitz"` default is also untested.

**Fix:** add scenarios covering each branch of the time-format reader,
including a record with no time-format fields at all (default → blitz),
and the explicit legacy `time_control` values that the switch handles.

### 1e. Pagination and ordering

- L68, L143 — `Math.Max → Math.Min` mutations on pagination guards.
- L69, L172 — `(page - 1) * pageSize` arithmetic.
- L173 — `offset >= all.Count ? [] : all.Skip(offset).Take(pageSize)`.
- L153 — `id is not null && !deduped.ContainsKey(id)`.

**Fix:** add list-paging tests with boundary inputs (page 0, page beyond
last page, offset exactly at count, duplicate IDs).

### 1f. PGN tag-header strip (top of file)

Lines 44–49. `lastBracket > 0 ? pgn[(lastBracket + 1)..] : pgn` and the
surrounding tag-extraction strings.

**Fix:** add PGN-import tests covering: PGN with no tag block, PGN with
a single `[FEN "..."]` tag, PGN with several tags, and a PGN with a tag
character (`]`) inside a comment (edge case for `lastBracket`).

### 1g. Statement removals on `await ... Async` calls

L74, L87, L107, L116, L329, L332, L333, L334, L340, L343, L344, L347 —
statement mutations to `;`. These typically indicate that a side-effecting
call (repo write, append) isn't observed by any assertion.

**Fix:** for each, identify whether the test path verifies the side effect.
The pattern is the same as user-service `LastInsertRequest` — capture the
`InsertRequest` / `UpdateRequest` in the test context and assert the body.

## Phase 2 — Add `AnalysisSessionService` tests

Once Phase 1 is green, remove these lines from `stryker-config.json`:
```
"!**/Domain/AnalysisSession.cs",
"!**/Services/AnalysisSessionService.cs"
```

Then add a Reqnroll feature suite for sessions, following the pattern in
`Features/ImportFromPgn.feature` and `StepDefinitions/AnalysisGameServiceSteps.cs`:
- Session creation (with and without existing session — second one must
  cancel the first per the design doc)
- Navigation (`current_index` bounds, validation, whatif clearing)
- Whatif move acceptance / rejection
- Analysis stream replay from cache + live updates + cancellation
- Bot mismatch startup check
- Whatif PGN export

Refer to `CLAUDE.md` "Sessions" and "Analysis engine stream" sections for the
full behavioural contract.

The expected end state: `AnalysisSession.cs` and `AnalysisSessionService.cs`
both hit 100% line/branch/method coverage AND produce a mutation score
≥ 95% in Stryker.

## Constraints

- Do **not** add `[ExcludeFromCodeCoverage]` to `AnalysisSessionService` or
  any of its current methods to "fix" the coverage gap — write the tests.
- Do **not** add files to the `mutate` ignore list beyond the ones already
  listed above; if a mutant is truly equivalent, prefer a tight test
  assertion or an `ignore-methods` rule scoped to the specific method name.
- Keep tests deterministic — do not test the fire-and-forget bot-move
  triggers (those are already `[ExcludeFromCodeCoverage]`).

## Out of scope

- Refactoring `AnalysisGameService` to make it more testable. The current
  design is already test-friendly via internals visibility.
- Touching the `Adapters/Postgres/` or `Adapters/Mongo/` code paths in
  database-service (they're a separate concern; see
  `TASK_database_service_upgrade_mongodb_driver.md`).
