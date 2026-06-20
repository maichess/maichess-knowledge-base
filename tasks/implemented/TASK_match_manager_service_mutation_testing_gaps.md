# Task: Close mutation testing gaps in maichess-match-manager-service

## Background

A first Stryker.NET run on `services/maichess-match-manager-service` reports
a mutation score of ~72%. After excluding the exception classes
(`Services/*Exception.cs`) whose message strings carry no testable behaviour,
74 surviving mutants remain across two files:

- `Services/MatchService.cs` — 63 surviving mutants (core business logic)
- `Grpc/MatchesGrpcService.cs` — 11 surviving mutants (gRPC handler)

The config lives at
`services/maichess-match-manager-service/MaichessMatchManagerService.Tests/stryker-config.json`.

One mutant in `Services/TimeFormatRegistry.cs` (`Presets.First → FirstOrDefault`
on the hard-coded `"5+0"` lookup) is genuinely equivalent and has already
been marked with `// Stryker disable once Linq` in source.

## Goal

Drive the mutation score on the non-excluded code to ≥ 95%.

## Phase 1 — `MatchService.cs` (63 mutants)

Run `cd MaichessMatchManagerService.Tests && dotnet stryker` and inspect
`StrykerOutput/<timestamp>/reports/mutation-report.html`. The categories
below cover the surviving mutants seen on 2026-05-27.

### 1a. Pagination defaults (L65–67)

`Math.Max(1, page)`, `Math.Min(pageSize, MaxPageSize)`, `category != "" ? category : null` —
boundary mutations on `page <= 1`, `pageSize <= 0`, `Min → Max`, and the
category-empty check survive.

**Fix:** add list-paging tests covering `page = 0`, `page = -1`, `pageSize = 0`,
`pageSize > MaxPageSize`, `category = ""`, `category = null`, and
`category = "blitz"`. Assert both the returned page and the parameters
passed to the repository's pagination query.

### 1b. White/black player ternaries

L137, L172, L186, L252 — `match.Turn == "white" ? match.White : match.Black`
style ternaries (or similar). Mutations to the condition or branches survive,
suggesting tests only cover one side.

**Fix:** for each method using a turn-dependent branch (`SubmitMoveAsync`,
`OfferDrawAsync`, `AcceptDrawAsync`, `ResignAsync`), parameterise the
existing scenarios so that each runs once with white-to-move and once with
black-to-move, and assert the affected player object.

### 1c. Clock / timeout logic (L142–146, L317, L403, L405)

- L142 `match.Status == "ongoing"` — string and equality mutations survive.
- L144 `WhiteTimeMs <= 0 || BlackTimeMs <= 0` — logical and boundary mutations.
- L145 `"timeout"` — end-reason string.
- L317 `elapsed > remainingMs` — clock-elapsed comparison.
- L403 `increment > 0` — increment guard.

**Fix:** add explicit clock tests:
- Submitting a move that depletes the mover's clock to exactly `0` → flag.
- Submitting on an already-ended match → rejected with the correct reason.
- A `TimeFormat` with `IncrementMs = 0` → mover's clock is NOT credited.
- A `TimeFormat` with `IncrementMs > 0` on a game-ending move → no increment.
- The exact `EndReason = "timeout"` string is emitted (not just any non-empty).

### 1d. Side-effect assertions (statement removals)

L99, L121, L135, L140, L184, L187, L220, L222, L250, L253, L278, L280, L334,
L335 — mutations from `something.Method(...)` to `;` survive, meaning the
test path does not verify the call.

**Fix:** the pattern matches what was applied in user-service for the
`InsertRequest` capture (see git log of `MaichessUserService.Tests/Support/GrpcServiceContext.cs`).
For each repository call (`UpdateAsync`, `InsertAsync`), socket-broadcast
call, and event publish, capture the request argument in the test context
and assert its fields.

### 1e. PGN export / stored-PGN strings (L421–426, L33)

L33 `"matchcreated"` style event constant.
L421–426 — PGN tag strings (`[Event "..."]`, `[Site "..."]`, …).

**Fix:** for PGN, add an export test that round-trips a known match and
asserts the exact PGN output as a multi-line string. For event-name
constants, assert that the broadcast event name matches the contract value
documented in `protos/socket-service/v1/socket.proto`.

### 1f. FEN parsing helpers (L432)

`parts.Length >= 2` mutation on FEN-segment count check.

**Fix:** add tests for malformed FENs (too few segments).

## Phase 2 — `MatchesGrpcService.cs` (11 mutants)

L53 `request.Page > 1 ? request.Page : 1` — page guard.
L79, L106, L145, L152 — string mutations (likely status codes / response field names).
L165, L171, L172 — TimeFormat null coalescing and string assignments.
L188 — statement removal.

**Fix:** the gRPC handler is supposed to be a thin proto ↔ domain translator
per `CLAUDE.md`. The MatchService tests are the right place for behaviour;
the gRPC tests should assert exact proto wire output for one example per
RPC, plus error-mapping. Add:
- A test per RPC that sends a request with default page/pageSize and asserts
  the proto response field-by-field.
- A test for `request.Page = 0` and `request.Page = -1` (kills L53).
- Error-mapping tests: domain exception → expected `RpcException.StatusCode`.

## Constraints

- Do **not** add `[ExcludeFromCodeCoverage]` to `MatchService` methods to
  "fix" the gaps — write tests.
- Do **not** disable individual mutations in source unless they are truly
  equivalent. The only legitimate `// Stryker disable` in this service so
  far is on `TimeFormatRegistry.Default`.
- Preserve the testing setup: Reqnroll BDD for MatchService business logic,
  plain xUnit for `MatchEventBroadcaster` and `MatchesGrpcService` per
  `CLAUDE.md`.

## Out of scope

- The fire-and-forget `TriggerBotMoveIfNeeded` / `ProcessBotMoveAsync` —
  already excluded with `[ExcludeFromCodeCoverage]`.
- `MatchRepository` (live MongoDB) — already excluded.
- REST endpoint adapters — already excluded.
