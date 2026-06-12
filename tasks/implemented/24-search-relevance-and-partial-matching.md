# 24 — Search: partial/substring matching, searchable names, bot games, clearer games-vs-matches

> Read `conventions.md` and
> [search-service.md](../../knowledge/services/search-service.md) first. Touches
> `maichess-search-service` (ES mappings + query builders + reindex) and the client
> `SearchPanel`. Builds on shipped task `13`.

## Reported problems (item 7)

1. **"What's the difference between games and matches?"** — Not surfaced in the UI.
   - **Games** = the analysis **games library** (`GET /search/games`): imported PGNs and
     match-derived analysis games (per-user).
   - **Matches** = **Past Matches** you played (`GET /search/matches`, per-user).
   - (The cross-user *All games* browser is a different feature — match-manager, task 07.)
   Fix: label/help-text in `SearchPanel` explaining each tab.
2. **Free-text doesn't find my own username** even though a game was played. Likely the
   index stores **player ids, not resolved usernames** ("best-effort id labels" per the
   design doc), and/or the searched field uses a `keyword`/exact analyzer. So a username
   query matches nothing.
3. **Can't find bot games.** Bot participants need their bot name/id indexed and included
   in the free-text query.
4. **Partial words/usernames don't match.** The text fields need an analyzer that supports
   substring/prefix matching, not just whole-token equality.

## Root-cause investigation (search-service)

- Inspect the ES index mappings shipped in task 13 (`maichess-search-service`,
  startup/init mappings) for the games and matches indexes. Identify the fields holding
  white/black player labels and whether they are `keyword` vs `text`, and whether
  **usernames/bot names are populated at all** by `CdcDocumentMapper` (the CDC →
  `IndexCommand` transform). If only ids are indexed, that is the primary bug.
- Confirm the query builders for `/search/games` and `/search/matches` include the
  player-name fields in the free-text query.

## Implementation

1. **Index searchable names.** Ensure `CdcDocumentMapper` writes both the human username
   **and** bot name (and ids) for white/black into analyzed text fields. If names aren't
   in the CDC payload, resolve them (or denormalize at index time) — document the approach.
2. **Partial matching.** Add an `edge_ngram` (prefix) or `ngram` (substring) analyzer for
   the name fields (or use `match_phrase_prefix` / a `search_as_you_type` field). Pick one
   approach, document the trade-off (index size vs recall), and apply consistently to
   games + matches name fields.
3. **Free-text query** covers: player usernames, bot names, opponent, opening/ECO, tags,
   PGN headers. Verify both endpoints.
4. **Reindex.** Mapping/analyzer changes require a reindex — run the existing one-shot
   reindex job (task 13) after the mapping ships. Note this in the rollout steps.
5. **Client `SearchPanel`:** add the games-vs-matches explanation and make sure bot-name
   queries are sent to the same free-text param.

## Testing

- Search-service: unit tests on `CdcDocumentMapper` (names now present) and the query
  builders (free-text includes name fields); analyzer behavior covered by an integration
  test if the harness allows, else documented manual ES verification. 100% on new
  non-excluded code; mirror exclusions into Stryker config.
- Client: `npm run build` + `npm run lint` + manual search by full and partial
  username/bot name.
