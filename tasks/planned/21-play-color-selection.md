# 21 — Play: choose your color (human queue + vs-bot)

> Read `conventions.md` first. Touches `maichess-api-contracts` (match-maker), the
> match-maker service, and `maichess-client`. Follow the contract publish/bump handoff.

## Goal

Let a player choose which color they play:

- **Find opponent (human matchmaking):** add a color-preference field — `white`,
  `black`, or `any` (default). The matchmaker pairs accordingly; two players who both
  want the same fixed color are still pairable (one gets their wish, or both fall back
  to `any` semantics — see Matching rules).
- **Play vs bot:** add a `white | black | random` toggle (default `random`) in the
  vs-bot setup. This one is mostly client-side: the client already creates the bot
  match directly, so it just needs to assign the human to the chosen color (or random).

## Background

- `app/play/page.tsx` + `MatchmakingModal.tsx` drive both flows. The dashboard links
  `?opponent=bot` for the bot flow.
- The human queue goes through the match-maker queue contract; the bot match is created
  via the match-maker bot-vs-human / create path. Read
  `maichess-api-contracts/proto/match-maker-service/v1/*` and
  `rest/match-maker.md` to find the queue-enqueue and bot-match-create messages
  **before** adding fields — do not infer from the client.

## Contracts

1. **Queue enqueue request:** add `ColorPreference color_preference = N;`
   ```protobuf
   enum ColorPreference {
     COLOR_PREFERENCE_UNSPECIFIED = 0; // treated as ANY
     COLOR_PREFERENCE_ANY = 1;
     COLOR_PREFERENCE_WHITE = 2;
     COLOR_PREFERENCE_BLACK = 3;
   }
   ```
2. **Bot match create request:** add the same enum (or a simpler `bool human_plays_white`
   plus a "random" sentinel) so the human's color is explicit. Prefer reusing
   `ColorPreference` for consistency.
3. Update `rest/match-maker.md`. Then **stop and prompt the user** to commit + tag
   (`vX.Y.Z`) + push contracts, and bump `Maichess.PlatformProtos` in every consumer
   (`.csproj` / `build.sbt`) to the published version.

## Matching rules (match-maker)

- Two `ANY` → assign colors as today.
- One side fixed, other `ANY` → fixed side gets its color.
- Both fixed **same** color → not directly compatible; either keep them queued for a
  better match or, after a short wait, coin-flip and pair anyway. Keep it simple:
  treat a same-color clash as compatible and coin-flip who concedes. Document the choice.
- Both fixed **opposite** colors → ideal pairing.

## Client

- `MatchmakingModal` / play page: add a color selector (segmented control: White /
  Any / Black) for the human queue, and a White / Random / Black toggle for vs-bot.
- Pass the selection through the existing enqueue/create hooks.
- Default both to "any"/"random" so current behavior is preserved when untouched.

## Testing

- Match-maker: unit tests for each matching-rule branch above (100% on new code).
- Client: `npm run build` + `npm run lint` + manual click-through (queue with each
  preference, vs-bot with each color).
