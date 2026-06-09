# Position History — Ownership and Lifecycle

## Problem

Detecting threefold repetition requires knowing every position that has occurred in the game.
The move validator operates on a single FEN per call and has no persistent state, so it cannot
detect repetitions on its own. Passing the full PGN on every call would work but grows linearly
with game length and gives the validator more data than it needs.

## Decision

`ValidateMoveRequest` carries a `position_history` field: an ordered list of **position keys**
for every position reached since the last pawn move or capture (the same window used by the
fifty-move clock, since repetition cannot span an irreversible move).

A position key is the first four space-separated fields of a FEN string:
`<pieces> <side-to-move> <castling> <en-passant>`. The half-move clock and full-move number are
excluded because they differ between repetitions and must not affect identity comparison.

The move validator is the **sole authority** over this list:

- It receives the list from the caller, appends the resulting position key when a move is valid,
  and returns the updated list in `ValidateMoveResponse.position_history`.
- When the move is a pawn move or capture (detected via `halfMoveClock == 0` in the resulting
  board) the validator resets the list to contain only the new position key, because no prior
  position can recur after an irreversible move.
- When a move is invalid the validator returns no updated list; the caller keeps the existing one.

The match manager **stores and relays** the list alongside game state but **never modifies it**.
It treats the list as an opaque blob whose meaning is entirely defined by the move validator.

## Lifecycle

| Event | Match Manager action |
|---|---|
| Game starts | Pass empty `position_history` on first `ValidateMoveRequest` |
| Valid move | Replace stored list with `position_history` from response |
| Invalid move | Keep existing stored list unchanged |
| Game ends (`game_result` ≠ `UNSPECIFIED` / `NONE`) | Discard the list — it is not needed after the game concludes |

## Why not PGN / why not a callback

Passing full PGN grows quadratically in total bytes over the game and leaks move-notation
concerns into the validator. A callback from validator → match manager would create a circular
service dependency. The position-history relay avoids both: the payload is bounded by the
fifty-move window (~100 FEN strings, ~8 KB worst case), the validator stays stateless, and the
call direction remains match-manager → move-validator only.

## Threefold repetition detection

The validator extracts the position key from the resulting FEN and counts its occurrences in the
incoming `position_history`. If the count is ≥ 2, the position now appears three times total and
the validator returns `GAME_RESULT_THREEFOLD_REPETITION` (the new occurrence is not appended —
the game is over). Because the history is bounded by irreversible moves, this check is only
performed when the resulting `halfMoveClock` is non-zero.

## Fifty-move rule

Detected from the FEN half-move clock field alone (`halfMoveClock >= 100`). The
`position_history` mechanism is not involved.
