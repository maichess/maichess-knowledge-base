# 01 — Dev-mode toggle

**Goal:** Add a persisted `dev_mode` flag to the user profile and use it to gate a
new "Dev" area in the client navigation. This is the foundational gate that
prompts `05` (and the Dev section generally) depend on.

> Read `conventions.md` first. Honor every convention there:
> contracts-first, the contract-versioning handoff, 100% coverage, mutation
> exclusions, Next.js 16 rules.

## Background to read first

- Contract: `maichess-api-contracts/protos/user-service/v1/users.proto`,
  `maichess-api-contracts/rest/users.md`.
- Service: `services/maichess-user-service/MaichessUserService/` — note that
  persistence goes through the **generic database-service CRUD** (a protobuf
  `Struct` record in the `users` collection, see `UsersService.cs`), not direct
  SQL. New profile fields are added as `Struct` fields, defaulted on create.
- Client: `lib/components/Nav.tsx`, `lib/constants/routes.ts`,
  `app/profile/page.tsx`, `app/profile/ProfileForm.tsx`, `lib/hooks/useProfile.ts`,
  `lib/models/user.ts`, `app/api/users/me/route.ts`.

## Contract changes (`maichess-api-contracts`)

1. In `users.proto`:
   - Add `bool dev_mode = 7;` to `User`.
   - Add a way to update it. Add `optional bool dev_mode = 3;` to
     `UpdateUserRequest` (so username and dev_mode can be patched independently —
     use a wrapper/`optional` so "not provided" is distinguishable from `false`).
2. In `rest/users.md`:
   - Add `dev_mode` to the `GET /users/me` response schema.
   - Add `dev_mode` (boolean, optional) to the `PATCH /users/me` request table.
3. **Publish + bump** per the versioning handoff in `00`: prompt the user to tag
   and push the contracts repo, then update the `Maichess.PlatformProtos` version
   in user-service’s `*.csproj` (and reconcile the other services to the same new
   version).

## Service changes (`maichess-user-service`)

- `UsersService.cs`:
  - On create, default `dev_mode` to `false` in the `Struct` record.
  - Read `dev_mode` in `UserFromStruct` (treat a missing field as `false` for
    pre-existing users — backward compatible, no data migration needed).
  - Extend `UpdateUserAsync` to accept an optional `dev_mode` and write it only
    when provided. Preserve the existing "username required / at least one field"
    semantics — now "at least one of username, dev_mode".
- `Grpc/UsersGrpcService.cs`: map the new request/response fields.
- `Rest/PatchUserRequest.cs`, `Rest/UserResponse.cs`, `Rest/UsersEndpoints.cs`:
  thread `dev_mode` through (endpoints/DTOs stay `[ExcludeFromCodeCoverage]`).
- Tests: extend the existing user-service suite to cover create-defaults-false,
  get-returns-dev_mode, patch-sets-true, patch-username-only-leaves-dev_mode,
  patch-dev_mode-only, and the missing-field-defaults-false path. Keep 100%
  line/branch/method coverage; update the Stryker config if new files appear.

## Client changes (`maichess-client`)

- `lib/models/user.ts`: add `dev_mode: boolean`.
- `lib/constants/routes.ts`: add a `dev` route (e.g. `/dev`).
- `app/profile/ProfileForm.tsx` + `lib/hooks/useProfile.ts`: add a "Developer
  mode" toggle that PATCHes `dev_mode` via `app/api/users/me/route.ts` (extend the
  proxy route to forward the field). Optimistic update + error handling matching
  the existing username-edit pattern.
- `lib/components/Nav.tsx`: when `user.dev_mode` is true, render a "Dev" nav entry
  pointing at the `dev` route (server component already fetches the user).
- **Route guard:** the `app/dev/` routes (created in `05`) must redirect
  non-dev users. Add the guard now as a shared helper (server-side check of
  `dev_mode`, redirect to dashboard if false) so `05` can drop pages straight in.
  Add a placeholder `app/dev/page.tsx` that uses the guard and shows an empty
  "Dev tools" shell, to be filled in by `05`.

## Verification

- `cd services/maichess-user-service && dotnet test -p:CollectCoverage=true` →
  100% on non-excluded code; `dotnet stryker` from the test project still healthy.
- `cd maichess-client && npm run lint && npm run build`.
- Manual: toggle Developer mode in Profile → the "Dev" entry appears/disappears in
  the nav; visiting `/dev` as a non-dev user redirects.

## Knowledge base

Add a short note under `maichess-knowledge-base/` recording that `dev_mode` is a
server-persisted profile flag (not client-only) and that it gates the client Dev
section.
