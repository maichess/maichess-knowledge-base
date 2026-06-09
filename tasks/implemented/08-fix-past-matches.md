# 08 — Fix: Past Matches list (Profile tab) not working

> Read `conventions.md` first. This is an independent bug-fix prompt;
> run it before the caching batch (`09`+). It touches match-manager and possibly the
> client proxy + user-service; no contract change is expected.

## Symptom

The **Past Matches** section under Profile (`maichess-client/app/profile/page.tsx` →
`MatchHistory` → `useMatchHistory` → `GET /api/users/me/matches`) shows nothing / an error,
while the rest of the app (playing, watching, stats) works.

## Root-cause analysis (already done — confirm, then fix)

The path is wired correctly end-to-end, so this is an **identity-representation mismatch**, not a
missing wire. The chain:

1. `app/api/users/me/matches/route.ts` resolves the user via user-service `GET /users/me`, reads
   `me.id`, then calls match-manager `GET /users/{me.id}/matches` with the bearer token.
2. match-manager `ListUserMatches` (`Rest/MatchesEndpoints.cs`) asserts
   `userId (path) == authUserId (JWT sub)` → else **403 Forbid**, then
   `MatchService.ListUserMatchesAsync` filters match docs by
   `white_user_id | black_user_id | created_by_user_id == userId` (`Data/MatchRepository.FindForUserAsync`).

The asymmetry that breaks **only** this endpoint:

- auth-service mints the JWT with **`sub = user.id`** (raw stored value)
  — `services/maichess-auth-service/src/routes/auth.ts` (`signAccessToken({ sub: user.id, … })`).
- user-service `GET /users/me` returns **`id = Guid.Parse(user.Id)`** (re-normalised to canonical
  lowercase form) — `services/maichess-user-service/.../Rest/UsersEndpoints.cs` `ToResponse`.

Every other match-manager endpoint derives identity **only** from the JWT and never cross-checks
it against the user-service `id`, so they are immune. Past Matches is the one place the two id
representations must be byte-identical for both (a) the `userId == authUserId` assertion and
(b) the `white_user_id == userId` Mongo equality filter. Any divergence in **case or format**
between the raw `user.id` (in match docs / JWT sub) and the `Guid.Parse`-normalised `me.id`
produces either a **403** or a **silently empty list** — the reported symptom.

## Step 1 — Reproduce and confirm the branch

1. Log in; open Profile. Capture the actual HTTP status and body of
   `GET /api/users/me/matches?status=ended&page=1&page_size=20`.
2. Compare three values for the **same** account:
   - JWT `sub` (decode the `access_token`),
   - user-service `GET /users/me` → `id`,
   - a match document's `white_user_id` / `black_user_id` in match-db.
3. Determine the failure mode:
   - **403** → the `userId == authUserId` assertion is failing on representation.
   - **200 with empty `matches` but non-empty for that user in match-db** → the equality filter
     misses because stored ids are in a different representation than `me.id`.
   Both have the same fix.

## Step 2 — Fix (normalise identity representation)

Pick the minimal fix that makes the three representations consistent; prefer **canonicalising at
the boundaries** over loosening comparisons:

- **Preferred:** make the stored and minted id representation match user-service's canonical form.
  Ensure ids are written/compared in one canonical form (e.g. lowercase hyphenated Guid) at:
  the JWT `sub` (auth-service), the `white_user_id`/`black_user_id`/`created_by_user_id` values
  match-manager persists, and the `me.id` the proxy forwards. Identity is one representation
  everywhere.
- If a data-format skew already exists in stored matches, normalise on read in
  `FindForUserAsync`/`IsForUser` **and** fix the write path so new matches are canonical.
- For the assertion specifically: compare ids canonically (e.g. parse both as Guid, or compare
  case-insensitively) in `ListUserMatches` so a pure casing difference cannot 403 a legitimate
  owner — but still fix the underlying representation so the DB filter also matches.

> Do **not** "fix" this by removing the ownership assertion or by making the proxy skip the id
> check — the authorization that a user only lists their own matches must remain.

Consider whether the redundant `/users/me` round-trip in the proxy is worth keeping: match-manager
already has the authenticated id from the JWT. If the contract path
`GET /users/{user_id}/matches` is retained, keep the proxy resolving `me.id`, but the canonical-id
fix above is what makes it correct. Document any decision in the service `CONTRACT_NOTES.md` only
if a contract nuance surfaces (none expected).

## Step 3 — Tests (mandatory)

- match-manager: add a regression test in the `ListUserMatches` feature/step suite proving that a
  user whose id differs **only** in representation from the stored `white_user_id` still gets
  their matches (covers both the assertion and the filter). Keep 100% coverage on touched
  non-excluded code; `MatchRepository`/`MatchesEndpoints` stay `[ExcludeFromCodeCoverage]`, so put
  the assertion-level coverage where the service logic lives.
- If auth-service or user-service id handling changes, add/adjust their unit tests accordingly.
- Run `dotnet test -p:CollectCoverage=true` for each touched .NET service.

## Step 4 — Verify

1. Profile → Past Matches lists ended games (played **and** bot-vs-bot games the user started, per
   [match-history-and-stats.md](../../knowledge/domain/match-history-and-stats.md)),
   newest-first, paged.
2. `GET /api/users/me/matches` returns 200 with populated `matches`, `total`, `page`, `page_size`.
3. A second account does not see the first account's matches (authorization intact).
4. `npm run build && npm run lint` in `maichess-client`.

## Checklist

- [ ] Reproduced; recorded 403-vs-empty branch and the three id representations.
- [ ] Canonicalised identity representation across mint/store/compare.
- [ ] Ownership authorization preserved.
- [ ] Regression test added; coverage verified.
- [ ] Manual click-through passes.
