# Task: Close mutation testing gaps in maichess-move-validator-service

## Background

A first Stryker4s run on `services/maichess-move-validator-service` reports
a mutation score of **73.87% (of total) / 75.93% (of covered code)**. The
config lives at `services/maichess-move-validator-service/stryker4s.conf`.
Latest report (HTML) is at
`target/stryker4s-report/<timestamp>/index.html`.

The surviving mutants cluster into two categories — error-message string
literals (largest source of noise; mostly real test gaps in the error path)
and chess-rule logic mutations (highest-value to kill).

## Surviving mutants by file (2026-05-27 baseline)

### `src/main/scala/maichess/movevalidator/domain/Fen.scala`

Lines 14, 15, 18, 19, 20, 21 (multiple), 30, 34, 42, 43, 47 (many), 48, 53 —
StringLiteral, ConditionalExpression, LogicalOperator, EqualityOperator,
MethodExpression mutations. These are FEN parsing / serialisation:
splitting on whitespace, validating segment count, joining segments back.

**Fix:** in `FenParserSpec` (and a new `FenSpec` if needed), assert the
exact error messages produced by `Fen.parse` for each malformed input
(too few segments, invalid side-to-move token, invalid castling field,
invalid en-passant square). Cover the round-trip path:
`Fen.parse(s).flatMap(Fen.serialize) == Right(s)` for the standard set of
canonical positions.

### `src/main/scala/maichess/movevalidator/domain/UciMove.scala`

Lines 12, 16 — StringLiteral mutations on `s"UCI move must be 4 or 5
characters..."` and `s"Invalid squares in UCI move..."`.

**Fix:** assert the exact error strings on the negative-path tests in
`UciMoveSpec` (or wherever they live), not just `isLeft`.

### `src/main/scala/maichess/movevalidator/pgn/PgnParser.scala`

Lines 7, 12, 19, 22, 40, 46 — string literals and condition mutations on
tag parsing.

**Fix:** add `PgnParserSpec` cases for: PGN with malformed tag block,
PGN with no movetext, PGN with `[FEN "..."]` only, PGN with trailing
comment. Assert exact tag-map output, not just success/failure.

### `src/main/scala/maichess/movevalidator/rules/FenParser.scala`

Lines 13, 14, 16, 20, 32, 36, 55, 59, 61, 66, 70, 73 — string mutations
and the `parts.length` / `parts(1)` `==` guard.

**Fix:** explicit FEN-parse error-message assertions, plus tests for the
NoCoverage branches (very-short FEN strings, missing fullmove number).

### `src/main/scala/maichess/movevalidator/rules/LegalityFilter.scala`

Lines 26, 31, 35–36, 48, 51 — BooleanLiteral, EqualityOperator,
ConditionalExpression, LogicalOperator mutations. This is the
"is the move legal" filter — high-value to kill.

**Fix:** add positions where each branch is the discriminator:
- King-in-check filter — a move that leaves the king in check vs. a move
  that doesn't, on the same starting position.
- Castling-through-check guard — castling through an attacked square.
- En-passant pin discovery — en-passant that exposes the king.
- Pin filter (the `kingSquare == ...` equality on L48).

### `src/main/scala/maichess/movevalidator/rules/MoveApplicator.scala`

Lines 36, 38, 39 (many), 42, 43, 50, 51, 56, 57, 61, 65, 73, 74, 76, 83
(four strings), 84, 87, 88 — BooleanLiteral, EqualityOperator,
ConditionalExpression, MethodExpression, StringLiteral, LogicalOperator.
Move-application logic: castling rights update, en-passant target
calculation, promotion piece placement, halfmove clock reset on
pawn-move / capture.

**Fix:** add round-trip tests for every move kind:
- King move clears both castling rights for that side.
- Single rook move clears only the rook's side castling right.
- En-passant capture removes both the capturing pawn's old square AND
  the captured pawn's square.
- Promotion places the chosen piece, not the pawn.
- Halfmove clock resets on pawn move and on capture; increments otherwise.
- Fullmove number increments after black's move only.

### `src/main/scala/maichess/movevalidator/rules/MoveGenerator.scala`

Lines 35, 39, 82 (six strings), 86 (multiple), 101 — string mutations on
square names (probably the algebraic-notation table) and a `BooleanLiteral`
on a generator branch.

**Fix:** add a test asserting the exact algebraic name for each of the 64
squares (`a1` through `h8`). For the L101 boolean, identify the branch
(likely "side to move filter") and add the opposite-side test.

### `src/main/scala/maichess/movevalidator/rules/Board.scala`

Lines 20, 30 — LogicalOperator mutations on `&&` / `||`. Likely the
piece-on-square + colour-match check.

**Fix:** for each branch, add a test where exactly one operand is false.

### `src/main/scala/maichess/movevalidator/grpc/MovesServiceImpl.scala`

Lines 28, 29, 45 — `BooleanLiteral` mutations marked `NoCoverage`. These
are the gRPC handler's `ValidateMoveResponse(valid = false, ...)` flags
on the error paths.

**Fix:** integration-test each RPC with a malformed request and assert
`response.valid == false` AND the response carries the correct reason
string.

## Goal

Drive the mutation score on covered code from 75.93% to ≥ 95%. The bulk
of the work is exact error-message assertions plus rule-correctness tests
in `LegalityFilter` and `MoveApplicator`.

## Constraints

- Do **not** add `excluded-mutations` entries to `stryker4s.conf` to silence
  these — the chess-rule mutants are the whole point of mutation testing
  this service.
- The service `CLAUDE.md` mandates 100% statement coverage on logic code;
  this task is the natural next step beyond line coverage.
- Use `zio-test` as already configured. New tests should follow the
  `ZIOSpecDefault` pattern used in the existing specs under
  `src/test/scala/maichess/movevalidator/`.

## Out of scope

- `Main.scala` (server entry point, already excluded in `stryker4s.conf`).
- Refactoring any of the rule modules. They're well-structured already;
  the gaps are in tests, not code.
