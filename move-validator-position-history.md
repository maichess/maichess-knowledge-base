# Position History — Ownership and Lifecycle

## Problem

Detecting threefold repetition requires knowing every position that has occurred in the game.
The move validator operates on a single FEN per call and has no persistent state, so it cannot
detect repetitions on its own. Passing the full PGN on every call would work but grows linearly
with game length and gives the validator more data than it needs.

## Decision

`ValidateMoveRequest` carries a `position_history` field: an ordered list of FEN strings for
every position reached since the last pawn move or capture (the same window used by the
fifty-move clock, since repetition cannot span an irreversible move).

The move validator is the **sole authority** over this list:

- It receives the list from the caller, appends the resulting position FEN when a move is valid,
  and returns the updated list in `ValidateMoveResponse.position_history`.
- When the move is a pawn move or capture the validator resets the list (returns it empty),
  because no prior position can recur after an irreversible move.
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

The validator checks whether the resulting position FEN appears in `position_history` at least
twice before appending it. If so, the position will appear three times total, and the validator
returns `GAME_RESULT_THREEFOLD_REPETITION`.

## Fifty-move rule

Detected from the FEN half-move clock field alone (`halfMoveClock >= 100`). The
`position_history` mechanism is not involved.
