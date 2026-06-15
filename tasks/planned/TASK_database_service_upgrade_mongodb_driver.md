# Task: Resolve MongoDB.Driver transitive vulnerabilities in maichess-database-service

## Background

`services/maichess-database-service` currently fails to build because the
project enables `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` and the
default `NuGetAuditMode` (which is `all`, covering transitive deps) reports:

- **NU1903** (high): `Snappier` 1.0.0 — https://github.com/advisories/GHSA-pggp-6c3x-2xmx
- **NU1902** (moderate): `SharpCompress` 0.30.1 — https://github.com/advisories/GHSA-6c8g-7p36-r338

Both are pulled in transitively by `MongoDB.Driver` 3.1.0
(see `services/maichess-database-service/MaichessDatabaseService.csproj`).

`dotnet build` and `dotnet test` therefore fail on a fresh checkout. The
workaround `-p:NuGetAuditMode=direct` makes the build pass but only hides the
warning — the vulnerable packages are still loaded at runtime.

This blocks the local Stryker.NET run for this service
(`services/maichess-database-service/MaichessDatabaseService.Tests/stryker-config.json`),
because Stryker invokes `dotnet build` internally and inherits the same
failure.

## Goal

Make `dotnet test maichess-database-service.sln` succeed on a clean checkout
without bypassing or suppressing the audit, and resolve both advisories at
their source.

## Suggested approach

1. **Bump `MongoDB.Driver`** in `MaichessDatabaseService.csproj` (currently
   3.1.0; latest at time of writing is 3.9.0). Verify that the chosen version
   ships transitive deps `Snappier >= 1.1.6` and a `SharpCompress` version
   that no longer matches the advisory.
2. If the bump introduces breaking changes in the `Adapters/Mongo/`
   implementation, update the adapter accordingly. The adapter is the only
   layer permitted to import MongoDB types (see service `CLAUDE.md`).
3. Update `MaichessDatabaseService.Tests.csproj` to the same MongoDB.Driver
   version (currently also pinned to 3.1.0).
4. Run `dotnet test maichess-database-service.sln` with the default audit
   mode and confirm a clean build + green tests.
5. After the build is green, run Stryker:
   ```powershell
   cd services\maichess-database-service
   dotnet tool restore
   cd MaichessDatabaseService.Tests
   dotnet stryker
   ```
   Inspect the HTML report under `StrykerOutput/<timestamp>/reports/`. Apply
   the same mutant-evaluation playbook used on the other services: kill
   real-test-gap mutants by tightening assertions; for clearly equivalent or
   infrastructure-only mutants, extend `stryker-config.json` `ignore-methods`
   or `mutate` exclusions.

## Constraints

- Do **not** add `<NoWarn>NU1902;NU1903</NoWarn>` or
  `<NuGetAuditMode>direct</NuGetAuditMode>` to the csproj as a permanent
  fix — the project's "warnings-as-errors" stance is intentional and these
  are real advisories.
- Do **not** disable `<TreatWarningsAsErrors>` either.
- The adapter pattern in `services/maichess-database-service/CLAUDE.md` must
  be preserved: only `Adapters/Mongo/` may reference MongoDB types.
- If a MongoDB.Driver upgrade breaks `Adapters/Postgres/` (it shouldn't —
  separate adapter), stop and document the surprise in `CONTRACT_NOTES.md`.

## Out of scope

- Upgrading `Npgsql` or any other unrelated dependency.
- Adding new features or refactoring beyond what the upgrade requires.
