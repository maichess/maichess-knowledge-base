# Dev mode

## Decision

`dev_mode` is a **server-persisted user-profile flag**, not a client-only
preference. It gates the client's "Dev" area (the developer section that hosts
the bot arena and game-browsing tools introduced by later features).

Persisting it server-side means the flag follows the user across devices and
sessions and can be enforced on the server (route guards), rather than being a
spoofable local toggle.

## Where it lives

- **Contract** (`maichess-api-contracts`):
  - `User.dev_mode` (`bool`, field 7) in
    `protos/user-service/v1/users.proto`.
  - `UpdateUserRequest.dev_mode` (`optional bool`, field 3) — `optional` so
    "not provided" is distinguishable from `false`, letting `username` and
    `dev_mode` be patched independently.
  - `dev_mode` in the `GET /users/me` response and the `PATCH /users/me`
    request (`rest/users.md`).
  - Introduced in `Maichess.PlatformProtos` v0.3.3.

- **User service** (`maichess-user-service`): stored as a field on the `users`
  `Struct` record via the generic database-service CRUD. Defaulted to `false`
  on create; read back with a missing-field-defaults-to-`false` fallback.
  `UpdateUserAsync` writes it only when provided ("at least one of username or
  dev_mode" must be present).

  > **`user-db` is PostgreSQL, not a schema-less store.** Every `Struct` field
  > the user service writes maps to a real, typed column. A `Struct` field with
  > no backing column makes the `UPDATE`/`INSERT` fail with `42703` (undefined
  > column), surfacing to the client as "Failed to update developer mode." The
  > read-side missing-field fallback only hides this on reads. Adding a profile
  > field therefore **requires a schema migration** in the database service
  > (`Adapters/Postgres/Migrations/UserPostgresMigration.cs`) — `dev_mode` and
  > the Glicko-2 columns (`rating`, `rating_deviation`, `volatility`) are added
  > there with idempotent `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`.

- **Client** (`maichess-client`):
  - `User.dev_mode` in `lib/models/user.ts`.
  - Toggle in `app/profile/ProfileForm.tsx` via `useProfile.setDevMode`
    (optimistic, PATCHes `{ dev_mode }` through `app/api/users/me`).
  - `lib/components/Nav.tsx` renders the "Dev" nav entry only when
    `user.dev_mode` is true.
  - **Server-side guard** `requireDevUser()` in `lib/utils/serverUser.ts`:
    non-authenticated → `/login`, authenticated non-dev → `/dashboard`. The
    `app/dev/*` routes call it; `/dev` is also in the middleware `PROTECTED`
    list. The nav entry is a convenience only — access is enforced server-side.
