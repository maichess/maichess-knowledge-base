# Implementation Prompt: Tier 4 — Enhanced Evaluation (King Safety, Pawn Structure, Full Mobility)

## Goal

Create `EvalV2` — a drop-in replacement for `Eval` with a richer static evaluation —
and wire it into a new `SearchV4` that uses it. The search algorithm is identical to
`SearchV3`; only the evaluation function changes. This isolates the ELO gain attributable
purely to evaluation quality.

Register nine new bots with variant `EngineVariant.EnhancedEval`.

**Prerequisite**: Tiers 0, 2, and 3 must already be applied.

## Codebase Context

Service: `services/maichess-engine-service/` — Scala 3 + ZIO 2.

Files to read before starting:
- `src/main/scala/maichess/engine/chess/Eval.scala` — current evaluation (copy and extend)
- `src/main/scala/maichess/engine/chess/SearchV3.scala` — copy for SearchV4
- `src/main/scala/maichess/engine/chess/Position.scala` — `pieceBB`, `occupied`, `kingSquare`,
  `sideToMove`, `mailbox`, `castling`
- `src/main/scala/maichess/engine/chess/BB.scala` — bitboard utilities (`lsb`, `clearLsb`,
  `popcount`, `nonEmpty`, file/rank masks)
- `src/main/scala/maichess/engine/chess/Attacks.scala` — `pawnAttacks`, `knightAttacks`,
  `kingAttacks`
- `src/main/scala/maichess/engine/chess/Magics.scala` — `rookAttacks`, `bishopAttacks`
- `src/main/scala/maichess/engine/chess/Types.scala` — `Col`, `PType`, piece constants

## Implementation

### New file: `chess/EvalV2.scala`

Copy `Eval.scala` verbatim, rename the object to `EvalV2`, then add the following terms.
The existing material + PST + mobility is preserved. New terms are added to the running
total returned by `evaluate`. Return value is still centipawns from side-to-move perspective.

Keep all new tables and weights as `private val` arrays or constants at the top of the object.

---

#### 1. Full Piece Mobility (Rooks and Queens)

The existing `Eval` gives mobility bonuses only to knights and bishops. Extend this to all
sliding pieces.

**Rook mobility**: `BB.popcount(Magics.rookAttacks(sq, occ))` — but exclude squares occupied
by your own pieces from the count (`& ~pos.colorBB(c)`). Bonus: 3 centipawns per square.

**Queen mobility**: Same with `Magics.rookAttacks | Magics.bishopAttacks`. Bonus: 1 centipawn
per square (queens are already mobile; over-rewarding causes premature queen development).

**Tapered**: Apply mobility bonuses tapered by game phase. In the endgame, rook and queen
mobility matters more. Scale rook bonus from 3 (opening) to 5 (endgame) using the existing
`phase` interpolation pattern.

---

#### 2. Pawn Structure

Compute a pawn file mask once per side: `pawnFiles(c)` is a 8-bit integer where bit `f`
is set if color `c` has a pawn on file `f`.

**Doubled pawns**: For each file, if two or more pawns of the same color exist, penalize
`-20` per doubled pawn (i.e., `-20 * (pawnCount(f) - 1)`). Use `BB.popcount(pawnsOnFile(f))`.

**Isolated pawns**: A pawn on file `f` is isolated if there are no friendly pawns on files
`f-1` or `f+1`. Penalty: `-15` per isolated pawn.

**Passed pawns**: A pawn on square `sq` (color White) is passed if there are no black pawns
on its file or adjacent files ahead of it. Formally, `blackPawns & passedMask(sq) == 0`,
where `passedMask(sq)` covers all squares on files `f-1..f+1` strictly above rank `sq/8`.
Precompute `passedMask` for all 64 squares for both colors.

Passed pawn bonus is tapered: higher rank = more dangerous. Use a rank bonus array:
```scala
private val PassedRankBonus = Array(0, 10, 20, 35, 55, 80, 120, 0) // rank 1..8
```
The bonus is `PassedRankBonus(rank) * (24 - phase) / 24` — passed pawns matter more in
the endgame when they can queen.

---

#### 3. King Safety

King safety applies only in the opening/middlegame (`phase > 8`). In the endgame the king
should be active, so king safety terms are tapered out as `phase` drops.

**King zone**: The 3×3 square centered on the king (clipped to the board). Compute as:
`kingAttacks(kingSq) | (1L << kingSq)` — the king's own square plus all adjacent squares.

**Attacker count**: For each enemy piece type, count how many of their attacks intersect
the king zone. Aggregate a threat score:

```scala
private val KingZoneThreat = Array(0, 2, 3, 3, 5, 10, 0)  // per piece type (Pawn..Queen)
```

The total threat score is `sum(KingZoneThreat(pt) * attackerCount(pt))`.
Apply this as a penalty to the defended king: `-(threatScore * phase) / 24`.

**Pawn shield**: Count pawns in the two ranks directly in front of the king on the king's
file and adjacent files. Each missing pawn is a penalty of `-10`. This only applies when
the king is on its back rank or one rank ahead (i.e., `kingSq.rank <= 1` for White after
normalising). Tapered: `shieldPenalty * phase / 24`.

**Open file penalty**: If there is no friendly pawn on the king's file, apply `-25`.
If the king's file has no pawn at all (open), apply `-40`. Tapered by phase.

---

#### 4. Rook Bonuses

**Rook on open file**: A file is open if it has no pawns of either color.
A file is semi-open if it has no friendly pawns but does have enemy pawns.
- Open file: `+20` per rook
- Semi-open file: `+10` per rook

**Rook on 7th rank** (rank 7 for White, rank 2 for Black): `+25` if the enemy king is on
the 8th rank (or on its back rank) — the rook is then most dangerous.

**Connected rooks** (same file or rank with no pieces between them): `+10` per pair.

All rook bonuses are tapered: multiply by `(phase + 12) / 24` so they scale down cleanly
into the endgame where rooks are naturally active.

---

### New file: `chess/SearchV4.scala`

Copy `SearchV3.scala` verbatim, rename the class to `SearchV4`, and replace the single
call to `Eval.evaluate(pos)` with `EvalV2.evaluate(pos)`. That is the only change.

### New Bots to Register

```
id: "eval_bullet"           name: "Eval Bullet"            elo: 1900  variant: EnhancedEval  timing: Fixed(100)
id: "eval_bullet_prop"      name: "Eval Bullet Prop"       elo: 1900  variant: EnhancedEval  timing: Proportional(40, 50, 100)
id: "eval_bullet_aggr"      name: "Eval Bullet Aggressive" elo: 1900  variant: EnhancedEval  timing: Aggressive(0.07, 50, 100)
id: "eval_blitz"            name: "Eval Blitz"             elo: 2200  variant: EnhancedEval  timing: Fixed(1000)
id: "eval_blitz_prop"       name: "Eval Blitz Prop"        elo: 2200  variant: EnhancedEval  timing: Proportional(30, 200, 1000)
id: "eval_blitz_aggr"       name: "Eval Blitz Aggressive"  elo: 2200  variant: EnhancedEval  timing: Aggressive(0.06, 200, 1000)
id: "eval_classical"        name: "Eval Classical"         elo: 2450  variant: EnhancedEval  timing: Fixed(5000)
id: "eval_classical_prop"   name: "Eval Classical Prop"    elo: 2450  variant: EnhancedEval  timing: Proportional(25, 500, 5000)
id: "eval_classical_aggr"   name: "Eval Classical Aggr"   elo: 2450  variant: EnhancedEval  timing: Aggressive(0.05, 500, 5000)
```

### Bot Descriptions

Each `BotConfig` carries a `description: String` field (added by the API change prompt).
Use the following values verbatim.

**eval_bullet**
"The ordering-tier search engine now paired with a richer positional evaluation: king safety (exposed kings are penalised), pawn structure (doubled, isolated, and passed pawns with rank-scaled bonuses), rook placement bonuses (open files, 7th rank), and full mobility evaluation for all pieces. All terms taper smoothly between opening and endgame. Tactically sharp and positionally aware — a challenging opponent even for strong club players. Fixed at 100 ms per move."

**eval_bullet_prop**
"Rich positional evaluation — king safety, pawn structure, rook bonuses, and full piece mobility — layered on top of the enhanced-ordering search. Proportional time allocation keeps its clock balanced across the game."

**eval_bullet_aggr**
"Rich positional evaluation layered on top of the enhanced-ordering search. Spends 7% of remaining time per move, thinking hardest in the opening and middlegame where positional understanding matters most."

**eval_blitz**
"With 1 second and a full positional model, this bot plays in a recognisably strategic way: it prefers castled kings, connected pawns, active rooks, and mobile pieces — not just short-term tactics. A significant step up from simpler tiers at the same time control."

**eval_blitz_prop**
"Full positional model with proportional time management. The bot adapts its thinking time to the game's rhythm, allocating more when positions are complex and less when play is straightforward."

**eval_blitz_aggr**
"Full positional model with an aggressive time fraction — invests 6% of remaining time per move. Plays its most deliberate, strategically rich chess in the opening and early middlegame."

**eval_classical**
"Given 5 seconds with the complete evaluation model, this bot plays at a strong amateur level. It handles king safety, pawn breaks, rook activation, and piece coordination in a coherent, strategic style. The strongest evaluation-based bot in the roster — tough for most club players."

**eval_classical_prop**
"Five seconds with the full positional evaluation, distributed proportionally. A deeply strategic bot that manages its time intelligently across the whole game."

**eval_classical_aggr**
"Five seconds with the full positional evaluation, spending 5% of remaining time per move. Plays its most strategically profound chess early, then simplifies efficiently as the endgame approaches."

### Update `EngineServiceLive`

```scala
case EngineVariant.EnhancedEval =>
  ZIO.attempt(new SearchV4().bestMove(pos, moveTimeMs))
     .mapError(e => s"Search failed: ${e.getMessage}")
```

## Testing Requirements

- Tests in `src/test/scala/maichess/engine/`; use `zio-test`
- `chess/` is excluded from coverage per service CLAUDE.md — write tests for correctness
- Write `EvalV2Spec`:
  - Doubled pawn: position with doubled pawns evaluates worse than equivalent position without
  - Isolated pawn: isolated pawn is penalized relative to connected pawn
  - Passed pawn: advanced passed pawn receives bonus; bonus increases with rank
  - King safety: position with open king evaluates worse than castled king, same material
  - Rook on open file: rook on open file scores higher than rook on closed file
  - Symmetry: the starting position evaluates to exactly 0 centipawns from either side
  - `EvalV2.evaluate` returns negative when down material, positive when up material
- Write `SearchV4Spec`:
  - Finds same best move as `SearchV3` on quiet positions where eval difference shouldn't matter
  - Correctly prefers keeping pawns connected over creating a doubled pawn when material is equal
  - Castles king-side rather than leaving king in center when opponent has attacking pieces
- Update `BotRegistrySpec`: new IDs resolve, variant is `EnhancedEval`

## Constraints

- Do not modify `Eval.scala`, `SearchV3.scala`, or any earlier tier — `EvalV2` and `SearchV4`
  are new files
- All new evaluation terms must be tapered using the existing `phase` variable (0–24 scale)
  so evaluation transitions smoothly between opening and endgame
- King safety terms must be scaled to zero when `phase <= 8` to avoid king-activity penalties
  in the endgame where an active king is correct play
- The passed-pawn mask precomputation must happen at object initialization (not per-call) —
  it is a 64-element array computable once at startup
- EvalV2 is a pure function with no mutable state — it is an `object`, not a `class`
