# Implementation Prompt: Tier 2 — Enhanced Search (LMR + NMP + PVS)

## Goal

Create a new search class `SearchV2` that extends the existing bitboard engine with three
search-level optimizations that are among the highest-leverage techniques in modern engines:

- **Late Move Reductions (LMR)** — the single largest search improvement
- **Null Move Pruning (NMP)** — major pruning of won positions
- **Principal Variation Search (PVS)** — faster re-search after the expected best move

Register nine new bots that use `SearchV2` with variant `EngineVariant.EnhancedSearch`.

**Prerequisite**: The architecture scaffolding from `prompt-tier0-basic-bot.md` must already be
applied (EngineVariant enum, BotConfig.variant field, EngineServiceLive dispatch method).

## Codebase Context

Service: `services/maichess-engine-service/` — Scala 3 + ZIO 2.

Files to read before starting:
- `src/main/scala/maichess/engine/chess/Search.scala` — copy this as the base for SearchV2
- `src/main/scala/maichess/engine/chess/Position.scala` — `makeMove`/`unmakeMove`, `sideToMove`
- `src/main/scala/maichess/engine/chess/LegalCheck.scala` — `isInCheck`
- `src/main/scala/maichess/engine/domain/BotRegistry.scala` — where to add new bots
- `src/main/scala/maichess/engine/service/EngineServiceLive.scala` — dispatch method to update

## Implementation

### New file: `chess/SearchV2.scala`

Start by copying `Search.scala` verbatim into `SearchV2.scala` and renaming the class.
Then apply the three techniques below, in order. Each must be gated so it does not fire
during quiescence search.

---

#### 1. Principal Variation Search (PVS)

**What it does**: After searching the first move at a node with a full `[alpha, beta]` window
(the expected best move), search all subsequent moves with a null window `[alpha, alpha+1]`.
If a subsequent move beats alpha (a surprise), re-search it with the full window to get the
exact score. This reduces work because most moves at a node are worse than the first.

**Where to apply**: In the main `negamax` loop, after `legal == 1` or later (i.e., after the
first legal move has been searched).

```scala
val sc =
  if legal == 1 then
    -negamax(pos, depth - 1, -beta, -alpha, ply + 1)
  else
    val zw = -negamax(pos, depth - 1, -alpha - 1, -alpha, ply + 1) // null-window
    if zw > alpha && zw < beta then
      -negamax(pos, depth - 1, -beta, -alpha, ply + 1)             // re-search
    else zw
```

---

#### 2. Late Move Reductions (LMR)

**What it does**: Moves searched late in the move list (high index) are likely worse. Reduce
their search depth by 1 (or more). If the reduced search beats alpha, re-search at full depth.
This is the single biggest practical improvement — it effectively doubles usable search depth.

**Conditions to apply LMR** (all must be true):
- `depth >= 3`
- `legal >= 4` (not one of the first few moves — those are more likely good)
- The move is a quiet move (not a capture, not a promotion)
- Not in check (`!LegalCheck.isInCheck(pos, pos.sideToMove)` *before* making the move)

**Reduction amount**: Start with a fixed reduction of 1 ply. A common formula is
`max(1, (Math.log(depth.toDouble) * Math.log(legal.toDouble) / 2.0).toInt)` for a
depth+move-count scaled reduction, but start simple and note it can be tuned.

**Integration with PVS**: Apply LMR inside the else branch (moves after the first):

```scala
val sc =
  if legal == 1 then
    -negamax(pos, depth - 1, -beta, -alpha, ply + 1)
  else if canLmr then
    val reduced = -negamax(pos, depth - 1 - reduction, -alpha - 1, -alpha, ply + 1)
    if reduced > alpha then
      -negamax(pos, depth - 1, -beta, -alpha, ply + 1)  // full re-search
    else reduced
  else
    val zw = -negamax(pos, depth - 1, -alpha - 1, -alpha, ply + 1)
    if zw > alpha && zw < beta then
      -negamax(pos, depth - 1, -beta, -alpha, ply + 1)
    else zw
```

---

#### 3. Null Move Pruning (NMP)

**What it does**: If the current position is so good that even passing your turn (making a
"null move") and letting the opponent respond still exceeds beta, prune the subtree. Catches
positions where you're ahead enough that no opponent response can refute it.

**Conditions to apply NMP** (all must be true):
- `depth >= 3`
- Not in check
- Not at the root (`ply > 0`)
- The position has non-pawn material for the side to move (avoids zugzwang in pure pawn
  endgames — check that at least one non-pawn, non-king piece exists for the side to move)
- No previous null move at this line (track with a `wasNullMove: Boolean` parameter threaded
  through `negamax`, or a stack array indexed by ply)

**Reduction**: Use `R = 3` (search at `depth - 1 - R`). Some engines use `R = 2 + depth/6`.

**Implementation**: Before the move loop, do:

```scala
if canNullMove then
  pos.makeNullMove()
  val nullScore = -negamax(pos, depth - 1 - R, -beta, -beta + 1, ply + 1, wasNullMove = true)
  pos.unmakeNullMove()
  if nullScore >= beta then return beta
```

**Null move on Position**: A null move just flips `sideToMove` and clears the en-passant
square (and updates the Zobrist hash accordingly). Add `makeNullMove()` and `unmakeNullMove()`
to `Position.scala`. These must save and restore `epSquare` and update `hash` correctly.

---

### New Bots to Register

```
id: "search_bullet"           name: "Search Bullet"           elo: 1600  variant: EnhancedSearch  timing: Fixed(100)
id: "search_bullet_prop"      name: "Search Bullet Prop"      elo: 1600  variant: EnhancedSearch  timing: Proportional(40, 50, 100)
id: "search_bullet_aggr"      name: "Search Bullet Aggressive"elo: 1600  variant: EnhancedSearch  timing: Aggressive(0.07, 50, 100)
id: "search_blitz"            name: "Search Blitz"            elo: 1900  variant: EnhancedSearch  timing: Fixed(1000)
id: "search_blitz_prop"       name: "Search Blitz Prop"       elo: 1900  variant: EnhancedSearch  timing: Proportional(30, 200, 1000)
id: "search_blitz_aggr"       name: "Search Blitz Aggressive" elo: 1900  variant: EnhancedSearch  timing: Aggressive(0.06, 200, 1000)
id: "search_classical"        name: "Search Classical"        elo: 2150  variant: EnhancedSearch  timing: Fixed(5000)
id: "search_classical_prop"   name: "Search Classical Prop"   elo: 2150  variant: EnhancedSearch  timing: Proportional(25, 500, 5000)
id: "search_classical_aggr"   name: "Search Classical Aggr"  elo: 2150  variant: EnhancedSearch  timing: Aggressive(0.05, 500, 5000)
```

### Bot Descriptions

Each `BotConfig` carries a `description: String` field (added by the API change prompt).
Use the following values verbatim.

**search_bullet**
"Adds three high-impact search techniques to the bitboard engine: Late Move Reductions cut depth on moves likely to be bad, Null Move Pruning prunes subtrees where even passing a turn beats the opponent, and Principal Variation Search confirms the best move with a narrow re-search window. Together they roughly double effective search depth at the same time budget. Fixed at 100 ms per move."

**search_bullet_prop**
"LMR, Null Move Pruning, and PVS layered on the bitboard engine roughly double effective search depth compared to the base tier. Allocates time proportionally — spreading the clock across estimated remaining moves so it never wastes it in a long game or rushes in a short one."

**search_bullet_aggr**
"LMR, Null Move Pruning, and PVS layered on the bitboard engine roughly double effective search depth. Spends 7% of remaining time per move, investing more in the opening where decisions carry the most weight."

**search_blitz**
"With 1 second and all three search enhancements active, this bot reaches depths where it begins to uncover complex middlegame tactics the base engine misses entirely. LMR, Null Move Pruning, and PVS together mean it wastes very little effort on bad moves."

**search_blitz_prop**
"LMR, Null Move Pruning, and PVS layered on the bitboard engine, with proportional time management. The bot adapts how long it thinks based on the game's progress, spending its clock where it matters most."

**search_blitz_aggr**
"LMR, Null Move Pruning, and PVS layered on the bitboard engine. Spends 6% of remaining time per move — strong and deliberate in the opening and middlegame, quicker as the endgame simplifies."

**search_classical**
"At 5 seconds per move with LMR, Null Move Pruning, and PVS active, this bot reaches search depths that would take the base engine several times longer to achieve. Expect sharp tactical awareness and consistent middlegame pressure — a challenging opponent for club-level players."

**search_classical_prop**
"At full depth with all three search enhancements, this bot invests its time proportionally — roughly 5 seconds per move in a standard game, with flexibility to think longer if fewer moves remain."

**search_classical_aggr**
"At full depth with LMR, Null Move Pruning, and PVS, this bot spends 5% of remaining time each move — front-loaded thinking in the opening, faster play as the position simplifies. Consistent tactical sharpness throughout the game."

### Update `EngineServiceLive`

Add `EnhancedSearch` branch to the dispatch method:

```scala
case EngineVariant.EnhancedSearch =>
  ZIO.attempt(new SearchV2().bestMove(pos, moveTimeMs))
     .mapError(e => s"Search failed: ${e.getMessage}")
```

## Testing Requirements

- Use `zio-test`; tests in `src/test/scala/maichess/engine/`
- `chess/` package is excluded from coverage per service CLAUDE.md — write tests anyway to
  verify correctness, but do not fail CI on coverage gaps in `SearchV2`
- Write `SearchV2Spec`:
  - Finds mate-in-1 in standard tactical positions (same positions used in SearchSpec if it exists)
  - Finds the same best move as `Search` on a set of test positions (regression: the techniques
    must not change the result, only the speed at which it is found)
  - Does not return `Move.None` from a position with legal moves
  - NMP does not fire in zugzwang-prone endgame positions (K+P vs K — verify it still finds
    the correct result, not that NMP didn't fire internally)
- Write `PositionSpec` additions: `makeNullMove`/`unmakeNullMove` correctly flips side and
  clears ep square; hash is consistent before and after
- Update `BotRegistrySpec`: all new bot IDs resolve; variant is `EnhancedSearch`

## Constraints

- Do not modify `Search.scala` — `SearchV2` is a new class
- Do not apply LMR to captures, promotions, or moves that give check
- Do not apply NMP when the side to move has only pawns and king (zugzwang risk)
- The `bestMoveAtDepth` and `extractPv` methods on `SearchV2` must work identically to
  `Search` (they are used by `analyzePosition`)
- Thread `wasNullMove` as a boolean parameter through `negamax` — do not use a field,
  as recursion would corrupt it for non-null branches
