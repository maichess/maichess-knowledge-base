# Implementation Prompt: Tier 3 — Move Ordering (Killers, History, SEE, Aspiration, Check Extensions)

## Goal

Create `SearchV3` building on `SearchV2` with five move-ordering and search-control
improvements that refine which moves get searched first and how deep they are searched.
Better ordering means more beta cutoffs, which means alpha-beta prunes more — the
benefits compound with LMR and NMP from Tier 2.

Register nine new bots with variant `EngineVariant.EnhancedOrdering`.

**Prerequisite**: Tiers 0 and 2 must already be applied (`EngineVariant`, `SearchV2`, dispatch).

## Codebase Context

Service: `services/maichess-engine-service/` — Scala 3 + ZIO 2.

Files to read before starting:
- `src/main/scala/maichess/engine/chess/SearchV2.scala` — base to copy from
- `src/main/scala/maichess/engine/chess/Search.scala` — original, for context on scoreM / pick
- `src/main/scala/maichess/engine/chess/Types.scala` — Move encoding: from/to/flag bits
- `src/main/scala/maichess/engine/chess/LegalCheck.scala` — `isInCheck`, `isLegal`
- `src/main/scala/maichess/engine/chess/Magics.scala` and `Attacks.scala` — for SEE
- `src/main/scala/maichess/engine/chess/Position.scala` — mailbox, piece types, occupied

## Implementation

### New file: `chess/SearchV3.scala`

Copy `SearchV2.scala` verbatim, rename the class, then apply the five techniques below.

---

#### 1. Killer Move Heuristic

**What it does**: When a quiet move causes a beta cutoff (it was "so good" that the opponent
won't let this position occur), remember it. At the same depth in sibling nodes, try it early
because it may cause another cutoff.

**Data structure**: Two killer slots per ply:
```scala
private val killers = Array.ofDim[Int](128, 2)  // killers(ply)(0|1)
```

**Update** (in the beta cutoff path, for quiet moves only):
```scala
if !Move.isCapture(mv) && !Move.isPromo(mv) then
  if killers(ply)(0) != mv then
    killers(ply)(1) = killers(ply)(0)
    killers(ply)(0) = mv
```

**Scoring** in `scoreM`: Assign killers priority between TT move and captures:
```scala
else if mv == killers(ply)(0) then 900000
else if mv == killers(ply)(1) then 800000
```

---

#### 2. History Heuristic

**What it does**: Track which (from-square, to-square) quiet moves have historically caused
beta cutoffs across the entire search. High-history moves are ordered earlier.

**Data structure**: `private val history = Array.ofDim[Int](2, 64, 64)` indexed by
`(sideToMove, from, to)`. Cap values at `1 << 20` to prevent overflow.

**Update** on beta cutoff by a quiet move:
```scala
history(pos.sideToMove)(Move.from(mv))(Move.to(mv)) += depth * depth
```

**Bonus for moves that didn't cause a cutoff** (history gravity / penalty):
```scala
history(pos.sideToMove)(Move.from(mv))(Move.to(mv)) -= depth  // for all quiet moves searched before the cutoff
```

**Scoring** in `scoreM` for quiet moves:
```scala
else history(pos.sideToMove)(Move.from(mv).toInt)(Move.to(mv).toInt)
```

The full priority order in `scoreM` is now:
1. TT move: 2,000,000
2. Captures by SEE (see below): 1,000,000 + SEE score (or MVV-LVA fallback)
3. Killer 0: 900,000
4. Killer 1: 800,000
5. Quiet moves: history score (can be negative)

---

#### 3. Static Exchange Evaluation (SEE)

**What it does**: For captures, simulate the full exchange sequence on a square to determine
whether the capture wins or loses material. Replace MVV-LVA capture scoring with SEE so
losing captures (e.g. taking a pawn with a queen when the pawn is defended) are ordered last.

**Implementation approach**:

```scala
private def see(pos: Position, to: Int, target: Int, from: Int, attacker: Int): Int
```

Use a "gain" array: `gain(0) = pieceValue(target)`. Then alternate sides, finding the least
valuable attacker of `to` in the remaining occupied bitboard (after removing `from`). Continue
until no attacker remains or the gain would be negative (the side can choose not to recapture).
Propagate gains back: `gain(d-1) = max(-gain(d), gain(d-1))`.

Use the piece value array `MV` already defined: `Array(100, 320, 330, 500, 900, 20000, 0)`.

**Scoring**: If `see(pos, ...) >= 0` the capture wins/is even — score `1,000,000 + see`.
If SEE is negative (losing capture) score it below quiet moves: `see - 100,000` (still
searchable, just deprioritized).

**Also use SEE for LMR gating**: Do not reduce moves where SEE is positive (they are likely
good captures worth searching at full depth).

---

#### 4. Aspiration Windows

**What it does**: In iterative deepening, start each new depth's search with a narrow window
around the previous iteration's score rather than `[-INF, INF]`. If the result falls outside
the window (a "fail"), widen it and re-search. Typically reduces the total nodes searched by
30–50% in practice.

**Where to apply**: In the iterative deepening loop inside `bestMove`:

```scala
var alpha = -INF; var beta = INF
var delta = 50  // initial window half-width in centipawns
if depth >= 4 && rootScore != 0 then
  alpha = rootScore - delta
  beta  = rootScore + delta

while true do
  negamax(pos, depth, alpha, beta, 0, false)
  val sc = rootScore
  if sc <= alpha then
    alpha = Math.max(alpha - delta, -INF); delta *= 2
  else if sc >= beta then
    beta = Math.min(beta + delta, INF); delta *= 2
  else
    break
```

Reset to `[-INF, INF]` when `depth < 4` (aspiration is not useful at shallow depths).

---

#### 5. Check Extensions

**What it does**: When the side to move is in check, extend the search depth by 1 at that
node. Chess combinations frequently hinge on check sequences; without extension the engine
truncates them prematurely.

**Where to apply**: At the top of `negamax`, before the depth == 0 check:

```scala
val inCheck = LegalCheck.isInCheck(pos, pos.sideToMove)
val extDepth = if inCheck then depth + 1 else depth
```

Then use `extDepth` as the effective depth for the depth == 0 test and recursive calls.
Cap total extensions so they cannot inflate depth unboundedly:
`val extDepth = if inCheck && ply < depth * 2 then depth + 1 else depth`

---

### New Bots to Register

```
id: "ordering_bullet"          name: "Ordering Bullet"           elo: 1750  variant: EnhancedOrdering  timing: Fixed(100)
id: "ordering_bullet_prop"     name: "Ordering Bullet Prop"      elo: 1750  variant: EnhancedOrdering  timing: Proportional(40, 50, 100)
id: "ordering_bullet_aggr"     name: "Ordering Bullet Aggressive"elo: 1750  variant: EnhancedOrdering  timing: Aggressive(0.07, 50, 100)
id: "ordering_blitz"           name: "Ordering Blitz"            elo: 2050  variant: EnhancedOrdering  timing: Fixed(1000)
id: "ordering_blitz_prop"      name: "Ordering Blitz Prop"       elo: 2050  variant: EnhancedOrdering  timing: Proportional(30, 200, 1000)
id: "ordering_blitz_aggr"      name: "Ordering Blitz Aggressive" elo: 2050  variant: EnhancedOrdering  timing: Aggressive(0.06, 200, 1000)
id: "ordering_classical"       name: "Ordering Classical"        elo: 2300  variant: EnhancedOrdering  timing: Fixed(5000)
id: "ordering_classical_prop"  name: "Ordering Classical Prop"   elo: 2300  variant: EnhancedOrdering  timing: Proportional(25, 500, 5000)
id: "ordering_classical_aggr"  name: "Ordering Classical Aggr"  elo: 2300  variant: EnhancedOrdering  timing: Aggressive(0.05, 500, 5000)
```

### Bot Descriptions

Each `BotConfig` carries a `description: String` field (added by the API change prompt).
Use the following values verbatim.

**ordering_bullet**
"Extends the enhanced search engine with five move-ordering improvements: killer moves remember tactics that recently caused cutoffs; history scores track which quiet moves are statistically strong; SEE-based capture ordering evaluates full exchange sequences rather than just piece values; aspiration windows narrow the search between iterations; and check extensions follow tactical check chains deeper. More pruning in the same time means more depth. Fixed at 100 ms per move."

**ordering_bullet_prop**
"Five move-ordering enhancements — killers, history heuristic, SEE captures, aspiration windows, and check extensions — make every node of the search count. Proportional time allocation keeps the clock balanced across the game."

**ordering_bullet_aggr**
"Five move-ordering enhancements — killers, history heuristic, SEE captures, aspiration windows, and check extensions — compound to reduce wasted search effort dramatically. Spends 7% of remaining time per move, investing most heavily in the opening and middlegame."

**ordering_blitz**
"With 1 second and all five ordering improvements active, this bot's search reaches depths where it reliably finds multi-move combinations that simpler tiers miss. Killer moves and history scores mean it rarely spends time on obviously bad moves."

**ordering_blitz_prop**
"All five move-ordering improvements with proportional time management. The bot adjusts its thinking time based on the game's progress, spending wisely across the expected remaining moves."

**ordering_blitz_aggr**
"All five move-ordering improvements with an aggressive 6% time fraction. Thinks hardest in the opening and middlegame where the ordering enhancements deliver the biggest benefit."

**ordering_classical**
"Given 5 seconds with all move-ordering enhancements, this is one of the strongest search-only bots in the roster. It rarely overlooks tactical opportunities and handles positional manoeuvring with notable consistency. A serious opponent for intermediate-to-advanced club players."

**ordering_classical_prop**
"Five seconds of enhanced-ordering search, distributed proportionally. This bot manages its clock carefully, ensuring it always has time to think without wasting it in simplified endgames."

**ordering_classical_aggr**
"Five seconds of enhanced-ordering search, spent at 5% of remaining time per move. Front-loads its strongest thinking into the critical phases of the game, then plays more quickly as positions simplify."

### Update `EngineServiceLive`

Add `EnhancedOrdering` branch:
```scala
case EngineVariant.EnhancedOrdering =>
  ZIO.attempt(new SearchV3().bestMove(pos, moveTimeMs))
     .mapError(e => s"Search failed: ${e.getMessage}")
```

## Testing Requirements

- Tests in `src/test/scala/maichess/engine/`; use `zio-test`
- `chess/` is excluded from coverage per service CLAUDE.md — write tests for correctness
- Write `SearchV3Spec`:
  - Killer moves: in a position with a repeated tactical theme, `SearchV3` finds the same
    best move as `SearchV2` (regression: ordering changes speed, not results)
  - SEE: test the `see` helper directly on known positions (e.g., queen takes defended pawn
    returns a negative score; rook takes undefended queen returns a positive score)
  - Check extension: in a position where correct play requires a check chain beyond the
    nominal depth, `SearchV3` finds the correct move and `SearchV2` (at the same time limit)
    does not — or use a fixed-depth comparison
  - Finds mate-in-2 and mate-in-3 reliably
  - Returns consistent results across two calls on the same position (determinism check)
- Update `BotRegistrySpec`: new IDs resolve, variant is `EnhancedOrdering`

## Constraints

- Do not modify `SearchV2.scala`
- Killers and history are per-`SearchV3` instance (fields on the class), not static — each
  request gets a fresh instance, so no cross-request contamination
- SEE must handle the en-passant capture correctly (the captured pawn is not on `to`)
- Do not apply check extensions recursively without the ply cap — uncapped extensions can
  cause exponential blowup in pathological positions
- History scores must be capped to prevent integer overflow after many searches
