# 20 — Bot arena matrix setup: color switching toggle

> Read `conventions.md` and
> [bot-arena-service.md](../../knowledge/services/bot-arena-service.md) first.
> Requires a `maichess-api-contracts` update (new enum field). Depends on `04` / `05`.

## Goal

Add a `ColorMode` field to the matrix collection setup so that users can choose how colors are
assigned for each pairing. Currently the matrix always plays each pairing twice, alternating
colors. Adding a `Random` mode (single game per pairing with randomly assigned colors) is useful
for quick exploratory series where color balance is not critical.

## Background

The single setup already has a `keepSwitchingColors` boolean (task 04 spec). The matrix setup
currently hard-codes "always alternating" — each pairing plays once as white, once as black. The
tournament setup pairs once per FEN with each bot playing both colors across rounds (different
mechanism). The matrix is the most useful target for this toggle.

## Two modes

| Mode | Behavior | Games per pairing per FEN |
|---|---|---|
| `AlternatingColors` (default) | Each pairing plays once as white and once as black (current behavior). | `2 × N` |
| `Random` | Each pairing plays once; colors are randomly assigned per game via the arena's existing `IArenaRandomProvider`. | `N` |

## Contracts

### 1. `maichess-api-contracts` — update the matrix setup proto

In `proto/bot-arena-service/v1/arena.proto` (or wherever the matrix setup request message is
defined), add:

```protobuf
enum MatrixColorMode {
  MATRIX_COLOR_MODE_UNSPECIFIED = 0;
  MATRIX_COLOR_MODE_ALTERNATING = 1;  // default — play twice, swap colors
  MATRIX_COLOR_MODE_RANDOM = 2;        // single game, random color assignment
}

// In the MatrixSetupRequest message:
MatrixColorMode color_mode = N;  // defaults to ALTERNATING on UNSPECIFIED
```

Follow the standard contract publish/bump handoff: commit + tag the contracts repo, then bump
`Maichess.PlatformProtos` in `maichess-bot-arena-service.csproj` and
`maichess-client/package.json`.

### 2. REST spec

Update `rest/bot-arena-service.md` to document the new `color_mode` field on the matrix setup
request body.

## Service (`maichess-bot-arena-service`)

- In `CollectionService.LaunchCollectionAsync` / the matrix expansion path, check `colorMode`:
  - `Alternating` (default): keep the existing behavior — for each pairing, emit two games with
    swapped colors.
  - `Random`: for each pairing, emit one game with `IArenaRandomProvider.NextBool()` determining
    which bot is white.
- The `ArenaGame` domain model and `ArenaStore` do not need changes — the color mode affects only
  game spawning, not game tracking.

## Client (`maichess-client`)

- In the matrix setup form/page, add a color mode toggle (similar to the existing single-setup
  `keepSwitchingColors` toggle).
- Default: `AlternatingColors`.
- Label suggestion: **"Color assignment"** → `Alternate (×2 per pairing)` / `Random (×1 per pairing)`.
- Update the result/standings view if it displays game count per pairing — the count will halve
  in `Random` mode.

## Tests (mandatory)

- `CollectionService` unit tests: add `colorMode = Random` and `colorMode = Alternating` cases
  for matrix launch — verify the correct number of game-spawn calls and color assignments.
- Use `NSubstitute` to mock `IArenaRandomProvider`; return a deterministic sequence.
- 100% coverage; run `dotnet test -p:CollectCoverage=true`.

## Verify

1. Create a matrix setup with `Random` mode; confirm each pairing spawns exactly `N` (not `2N`)
   games with randomly assigned colors.
2. Create a matrix setup with default/`Alternating` mode; confirm each pairing spawns `2N` games
   with swapped colors.
3. `npm run build` and `npm run lint` pass for the client.

## Checklist

- [ ] `MatrixColorMode` enum added to `arena.proto`; contracts repo tagged + published.
- [ ] `Maichess.PlatformProtos` bumped in `maichess-bot-arena-service.csproj`.
- [ ] `rest/bot-arena-service.md` updated with `color_mode` field.
- [ ] `CollectionService` matrix expansion updated to branch on `colorMode`.
- [ ] Client form: color mode toggle added to matrix setup page.
- [ ] Tests: both modes covered, `IArenaRandomProvider` mocked.
- [ ] `dotnet test -p:CollectCoverage=true` passes.
- [ ] `npm run build && npm run lint` passes.
