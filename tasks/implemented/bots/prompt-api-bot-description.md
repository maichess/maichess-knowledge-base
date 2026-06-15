# Implementation Prompt: Add Bot Description Field — Full Stack API Change

## Goal

Add a `description` field to the bot data model and propagate it through every layer of the
stack so the web UI can display a rich explanation of each bot's algorithm and character.

This is a cross-cutting change spanning five systems:

1. **Proto contract** (`maichess-api-contracts`) — add the field to the source of truth
2. **Engine service** (Scala) — add the field to `BotConfig`, populate it in `BotRegistry`,
   expose it through the gRPC handler
3. **Match Maker service** (C#) — pass the field through the `GET /bots` REST endpoint
4. **REST contract doc** (`maichess-api-contracts/rest/match-maker.md`) — update the spec
5. **Client** (Next.js / React) — extend the type model and render the description in the UI

**Implement in this order** — each layer depends on the one above.

---

## Layer 1 — Proto Contract

**File:** `maichess-api-contracts/protos/engine-service/v1/bots.proto`

Add field 4 to the `Bot` message:

```protobuf
message Bot {
  string id          = 1;
  string name        = 2;
  int32  elo         = 3;
  string description = 4;
}
```

After editing, regenerate the stubs:
- **Scala** (ScalaPB): stubs regenerate automatically on `sbt compile` — just run it to
  verify the generated code compiles.
- **C#** (Grpc.Tools): stubs regenerate automatically on `dotnet build` in the match-maker
  project — run it to verify.

Do not modify any other message in the proto file.

---

## Layer 2 — Engine Service (Scala)

Service root: `services/maichess-engine-service/`

### 2a. `domain/BotConfig.scala`

Add `description: String` as a required constructor field:

```scala
final case class BotConfig(
  id:          String,
  name:        String,
  elo:         Int,
  strategy:    TimingStrategy,
  variant:     EngineVariant = EngineVariant.Base,
  description: String,
)
```

### 2b. `domain/BotRegistry.scala`

Add descriptions to every existing bot. The Tier 0–5 tier prompts each specify descriptions
for their own bots. The existing nine base-tier bots need the following descriptions:

```
id: "bullet"
description: "A fully-featured bitboard engine that represents the board as packed 64-bit integers and analyses all pieces simultaneously using hardware instructions. A transposition table ensures positions are never evaluated twice, and captures are followed beyond the main search depth to prevent horizon blunders. Allocates a flat 100 ms per move — fast and instinctive."

id: "bullet_proportional"
description: "A fully-featured bitboard engine with transposition table and quiescence search. Distributes its remaining time proportionally across the estimated moves left in the game — a more clock-aware approach than a fixed allocation."

id: "bullet_aggressive"
description: "A fully-featured bitboard engine with transposition table and quiescence search. Spends 7% of remaining time per move, front-loading calculation into the early game where move choices are most consequential."

id: "blitz"
description: "A fully-featured bitboard engine using magic bitboards for instant sliding-piece attack lookup and a transposition table to avoid repeated work. With 1 second per move it reaches depths where it reliably handles basic tactics and applies positional pressure. A solid club-player level opponent."

id: "blitz_proportional"
description: "A fully-featured bitboard engine at blitz speed. Proportional time management means it invests its clock across the whole game, allocating roughly equal time per phase rather than rushing in the endgame."

id: "blitz_aggressive"
description: "A fully-featured bitboard engine at blitz speed. Uses 6% of remaining time per move — thinking most in the opening and middlegame, then playing quicker as the position simplifies."

id: "classical"
description: "The bitboard engine at its most patient: 5 seconds per move allows significantly deeper searches than the blitz variant. Capable of finding multi-move combinations and applying coherent positional pressure across the whole game. A strong club-level opponent."

id: "classical_proportional"
description: "The bitboard engine at classical pace with proportional time management. Adapts how long it thinks based on game progress — a thoughtful, clock-aware engine that never rushes the endgame."

id: "classical_aggressive"
description: "The bitboard engine at classical pace. Spends 5% of remaining time per move — a front-loaded approach that ensures its deepest thinking happens in the most critical phases of the game."
```

### 2c. `grpc/BotsServiceImpl.scala`

The `toProtoBot` helper currently maps `id`, `name`, and `elo`. Add `description`:

```scala
private def toProtoBot(config: BotConfig): ProtoBot =
  ProtoBot(
    id          = config.id,
    name        = config.name,
    elo         = config.elo,
    description = config.description,
  )
```

### 2d. Tests (Engine Service)

- Update `BotRegistrySpec`: verify that `BotRegistry.find("bullet").map(_.description)` is
  `Some(nonEmpty string)` for all nine existing bots. Add the same assertion for any new bots
  introduced by tier prompts that have already been applied.
- Update `BotsServiceSpec`: the `ListBots` response includes a non-empty `description` in
  each returned `Bot` proto message.
- Run `sbt test -p:CollectCoverage=true` and confirm all logic-containing code is covered.

---

## Layer 3 — Match Maker Service (C#)

Service root: `services/maichess-match-maker-service/`

**File:** `Rest/QueueEndpoints.cs`

### 3a. Extend `BotResponse`

```csharp
internal sealed record BotResponse(string Id, string Name, int Elo, string Description);
```

### 3b. Update the mapping from proto to `BotResponse`

Find the existing mapping (the `.Select(b => new BotResponse(...))` expression in the `GetBots`
handler) and add the `Description` field:

```csharp
new BotResponse(b.Id, b.Name, b.Elo, b.Description)
```

The JSON serialiser uses `JsonNamingPolicy.SnakeCaseLower`, so `Description` serialises as
`"description"` automatically — no further changes needed.

### 3c. Tests (Match Maker)

Find the existing test(s) that assert on the `GET /bots` response shape and add an assertion
that each bot object in the `bots` array contains a non-empty `"description"` string. If no
such test exists yet, write one using the project's existing test infrastructure pattern.

---

## Layer 4 — REST Contract Documentation

**File:** `maichess-api-contracts/rest/match-maker.md`

Find the `GET /bots` response example and add the `description` field:

```markdown
## GET /bots

Returns all available bots.

**Auth:** None

**200 OK**
```json
{
  "bots": [
    {
      "id": "blitz",
      "name": "Blitz",
      "elo": 1700,
      "description": "A fully-featured bitboard engine..."
    }
  ]
}
```
```

The description value in the example can be abbreviated with `...` — it is illustrative only.

---

## Layer 5 — Client (Next.js / React)

Client root: `maichess-client/`

### 5a. `lib/models/bot.ts`

Add the field to the interface:

```typescript
export interface Bot {
  id:          string
  name:        string
  elo:         number
  description: string
}
```

### 5b. `app/api/bots/route.ts`

This file proxies requests to the match-maker. No change needed — the proxy forwards the
full JSON body as-is, so `description` will pass through automatically.

### 5c. `app/play/page.tsx` — Display the description in the bot selector

The bot selector grid currently shows `bot.name` and `bot.elo` per button. Add the
description below the ELO in each card so the player can read it before selecting.

Find the bot button rendering block (the grid of bot cards inside the
`opponentType === 'bot'` conditional) and extend each card to show the description.
The layout should follow the existing design conventions in the file — do not introduce new
design libraries or change the overall page structure. A compact truncated preview with the
full text accessible (via a tooltip, an expand toggle, or simply wrapping text depending on
what fits the existing design) is fine; use whichever approach matches the UI style already
present.

### 5d. Tests (Client)

If the project has component tests or E2E tests that render the bot selector, update them to
assert that the description text appears in the rendered output for at least one bot. If no
such test exists, note it in `CONTRACT_NOTES.md` for the client but do not add a new testing
framework.

---

## Definition of Done

- [ ] `bots.proto` has `string description = 4;`
- [ ] `sbt compile` succeeds in the engine service
- [ ] `dotnet build` succeeds in the match-maker service
- [ ] `BotRegistry.all` has a non-empty `description` on every bot
- [ ] `GET /bots` response (via curl or test) includes `"description"` for each bot
- [ ] The client `Bot` interface has `description: string`
- [ ] The bot selector in `/play` displays the description text
- [ ] All existing tests pass; new assertions added per the requirements above

## Constraints

- Do not change field numbers 1–3 in the `Bot` proto message — proto field numbers are wire
  format identifiers and changing them breaks compatibility with deployed services
- The `description` field is mandatory in `BotConfig` — do not give it a default value.
  Every bot must explicitly declare its description; a missing description should be a
  compile error, not a silent empty string
- Do not modify the `BotsListResponse` wrapper — only the `Bot` message changes
- Do not change the JSON key name: it must be `"description"` (snake_case, lowercase) to
  match the existing serialisation policy in the match-maker's `Program.cs`
- Do not alter the `POST /queue` endpoint — `bot_id` is the only bot field sent by the client
  when joining a queue; the description is display-only
