# Implementation Prompt: Tier 0 — Basic Mailbox Bot + Engine Variant Architecture

## Goal

Two things in one task:

1. **Architecture scaffolding**: add an `EngineVariant` enum to `BotConfig` and update
   `EngineServiceLive` to dispatch to the right engine implementation per bot. This is a
   prerequisite for all future tiers.

2. **Tier 0 engine**: implement a completely separate, simple chess engine using a plain
   8×8 int-array (mailbox) board — no bitboards, no magic numbers, no Zobrist hashing.
   Register it as three new bots. This tier exists as an educational baseline to show how
   primitive a raw minimax looks compared to the optimized bitboard engine.

## Codebase Context

Service: `services/maichess-engine-service/` — Scala 3 + ZIO 2, sbt build.

Key files to read before starting:
- `src/main/scala/maichess/engine/domain/BotConfig.scala` — currently has id, name, elo, strategy
- `src/main/scala/maichess/engine/domain/BotRegistry.scala` — list of all registered bots
- `src/main/scala/maichess/engine/service/EngineServiceLive.scala` — always calls `new Search()`
- `src/main/scala/maichess/engine/chess/Search.scala` — the existing bitboard engine
- `src/main/scala/maichess/engine/chess/Position.scala` — bitboard Position (FEN parser)
- `src/main/scala/maichess/engine/chess/Eval.scala` — existing static evaluation

Also read the service `CLAUDE.md` for conventions (ZIO style, WartRemover, coverage rules).

## Part 1 — Architecture: EngineVariant

### New file: `domain/EngineVariant.scala`

```scala
package maichess.engine.domain

enum EngineVariant:
  case Basic           // Tier 0: mailbox minimax, no bitboards
  case Base            // Tier 1: bitboard alpha-beta (current Search.scala)
  case EnhancedSearch  // Tier 2: + LMR, null-move pruning, PVS
  case EnhancedOrdering// Tier 3: + killer moves, history, SEE, aspiration, check extensions
  case EnhancedEval    // Tier 4: + king safety, pawn structure, full mobility
  case Knowledge       // Tier 5: + opening book, endgame tablebase probing
```

### Modify: `domain/BotConfig.scala`

Add `variant: EngineVariant = EngineVariant.Base` as a field. Existing bots get the default
automatically; no migration needed.

### Modify: `domain/BotRegistry.scala`

Update all existing `BotConfig(...)` calls to pass `variant = EngineVariant.Base` explicitly
(makes the dispatch visible at the call site). Then add the new Tier 0 bots (see below).

### Modify: `service/EngineServiceLive.scala`

Replace the hardcoded `new Search().bestMove(pos, moveTimeMs)` with a dispatch:

```scala
private def runSearch(fen: String, config: BotConfig, moveTimeMs: Long): IO[String, (String, Int)]
```

For `Basic`, parse FEN into a `BasicPosition` and run `BasicSearch`. For all others, parse
into the bitboard `Position` and run `Search`. Future tiers will add branches here as new
`Search` subclasses are created.

The `analyzePosition` path only applies to `Base` and above; return an error for `Basic`.

## Part 2 — Tier 0 Engine

### New package: `chess/basic/`

Four files. Keep everything self-contained — do not import anything from `chess/`.

#### `BasicPosition.scala`

- Board: `val board = new Array[Int](64)` using constants for pieces
  (`Empty=0, WPawn=1, WKnight=2, WBishop=3, WRook=4, WQueen=5, WKing=6`,
  and `BPawn=-1 .. BKing=-6`)
- State fields: `var sideToMove: Int` (1=White, -1=Black), `var castling: Int` (4 bits),
  `var epSquare: Int` (-1 = none), `var halfmove: Int`, `var fullmove: Int`
- `def fromFen(fen: String): Either[String, BasicPosition]` — parse the six FEN fields
- `def makeMove(mv: BasicMove): Unit` and `def unmakeMove(mv: BasicMove, saved: BasicState): Unit`
- `BasicState` is a simple value class capturing all mutable fields for unmake

#### `BasicMove.scala`

Plain case class: `case class BasicMove(from: Int, to: Int, promo: Int = 0, flag: Int = 0)`

Flags: `Quiet=0, DoublePush=1, Castle=2, EP=3, Capture=4`. `promo` encodes piece type
(0 = none, use piece constants above).

#### `BasicMoveGen.scala`

Generate pseudo-legal moves by iterating all 64 squares, checking piece type, and computing
target squares with direction arrays. Sliding pieces loop along rays until blocked.
Separate methods: `generateAll(pos)` and `generateCaptures(pos)`.

Filter to legal moves by making the move, calling `isInCheck(pos, side)`, and unmaking.
`isInCheck` loops all squares looking for the king square, then scans for attacking pieces.

No magic bitboards. No bitwise tricks. This code should be obviously readable.

#### `BasicSearch.scala`

- Minimax with alpha-beta pruning, iterative deepening, time limit via `System.currentTimeMillis()`
- No transposition table
- Move ordering: captures before quiet moves only (no MVV-LVA scores needed, just partition)
- Evaluation: call `BasicEval.evaluate(pos)` (see below)
- Check time every 512 nodes
- Interface: `def bestMove(pos: BasicPosition, timeLimitMs: Long): (BasicMove, Int)`
  Returns `(null, 0)` if no legal move exists.
- Return UCI string via a helper: `def toUci(mv: BasicMove): String`

#### `BasicEval.scala`

Material only: `Pawn=100, Knight=320, Bishop=330, Rook=500, Queen=900`.
Sum material for the side to move minus the opponent. No PST, no mobility.

### New Bots to Register

```
id: "basic_bullet"            name: "Basic Bullet"            elo: 700   variant: Basic  timing: Fixed(100)
id: "basic_bullet_prop"       name: "Basic Bullet Prop"       elo: 700   variant: Basic  timing: Proportional(40, 50, 100)
id: "basic_bullet_aggr"       name: "Basic Bullet Aggressive" elo: 700   variant: Basic  timing: Aggressive(0.07, 50, 100)
id: "basic_blitz"             name: "Basic Blitz"             elo: 800   variant: Basic  timing: Fixed(1000)
id: "basic_blitz_prop"        name: "Basic Blitz Prop"        elo: 800   variant: Basic  timing: Proportional(30, 200, 1000)
id: "basic_blitz_aggr"        name: "Basic Blitz Aggressive"  elo: 800   variant: Basic  timing: Aggressive(0.06, 200, 1000)
id: "basic_classical"         name: "Basic Classical"         elo: 900   variant: Basic  timing: Fixed(5000)
id: "basic_classical_prop"    name: "Basic Classical Prop"    elo: 900   variant: Basic  timing: Proportional(25, 500, 5000)
id: "basic_classical_aggr"    name: "Basic Classical Aggr"    elo: 900   variant: Basic  timing: Aggressive(0.05, 500, 5000)
```

### Bot Descriptions

Each `BotConfig` carries a `description: String` field (added by the API change prompt).
Use the following values verbatim.

**basic_bullet**
"A bare-bones engine that sees chess purely as material on a board. No bitboards, no position memory, no move ordering — just brute-force minimax with plain material counting. Allocates a flat 100 ms per move, making every decision as fast and honest as possible."

**basic_bullet_prop**
"A bare-bones engine that sees chess purely as material on a board. No bitboards, no position memory, no move ordering — just brute-force minimax with plain material counting. Divides remaining time proportionally across expected moves remaining so it never burns the clock early."

**basic_bullet_aggr**
"A bare-bones engine that sees chess purely as material on a board. No bitboards, no position memory, no move ordering — just brute-force minimax with plain material counting. Spends 7% of remaining time per move, front-loading calculation into the opening and thinning out as the game nears its end."

**basic_blitz**
"The simplest engine in the roster — a plain 8×8 board array with material-only evaluation and no position memory. Without a transposition table it re-analyses the same positions repeatedly, but 1 second per move lets it reach a few extra plies of depth, making it a small but visible step up in tactical sharpness."

**basic_blitz_prop**
"The simplest engine in the roster — a plain 8×8 board array with material-only evaluation and no position memory. Spreads its clock evenly across the game, adapting move time to how many moves it estimates are left."

**basic_blitz_aggr**
"The simplest engine in the roster — a plain 8×8 board array with material-only evaluation and no position memory. Uses 6% of remaining time per move — a middle ground between consistent pacing and front-loaded thinking."

**basic_classical**
"The bare-bones engine at its most patient: 5 seconds per move lets it reach its deepest searches, but the lack of position memory and primitive evaluation still cap its strength well below club level. A good starting opponent for beginners or anyone curious about how much modern optimisations actually matter."

**basic_classical_prop**
"The bare-bones engine at its most patient. Distributes its clock proportionally — in a typical game it invests roughly 5 seconds per move, with the flexibility to think longer if fewer moves remain, avoiding any risk of running out of time."

**basic_classical_aggr**
"The bare-bones engine at its most patient. Spends 5% of remaining time each move, which at the start of a fresh game amounts to about 5 seconds — comparable to the fixed variant but with a natural taper as the clock runs down."

## Testing Requirements

- Tests live in `src/test/scala/maichess/engine/`; use `zio-test` (`ZIOSpecDefault`)
- The `chess/basic/` package is **not** excluded from coverage — target 100% line/branch
- Write `BasicPositionSpec`: test FEN parsing (starting position, mid-game FEN, invalid FEN)
- Write `BasicMoveGenSpec`: starting position generates 20 moves; known tactical positions
- Write `BasicSearchSpec`: finds checkmate in 1 (e.g. "r1bqkb1r/pppp1Qpp/2n2n2/4p3/2B1P3/8/PPPP1PPP/RNB1K1NR b KQkq - 0 4"); finds only legal move; returns no move when stalemated
- Write `BasicEvalSpec`: evaluate starting position returns 0; material advantage detected
- Write `BotRegistrySpec` (extend existing): all new bot IDs found; variants set correctly
- Write `EngineServiceSpec` (extend existing): `bestMove` dispatches to basic engine for `basic_bullet`
- Run `sbt test -p:CollectCoverage=true` and verify all new code is covered before finishing

## Constraints

- Do not import anything from `maichess.engine.chess.*` into `chess/basic/`
- Do not add a transposition table — that is Tier 1's advantage and must remain visible
- Keep `BasicSearch` readable: a student seeing minimax for the first time should understand it
- WartRemover: suppress `Wart.Var` and `Wart.Return` locally in `BasicSearch` and `BasicMoveGen`
  where mutable state is unavoidable (same pattern as existing `Search.scala`)
- Do not modify the gRPC proto or the `BotsServiceImpl` beyond what is needed for dispatch
