# Task Specs — Overview & Conventions

This is the home for maichess **implementation task specs**. Each `NN-*.md` file is a
self-contained, runnable prompt for one feature or migration, executed as its own working
session. They are filed by status:

- **`implemented/`** — specs whose work has shipped (kept as the historical record of *what* was
  built and *why*).
- **`planned/`** — specs not yet started, or only partly built.

> **Status board:** [ROADMAP.md](ROADMAP.md) is the single source of truth for the task list,
> each task's status, and the dependency order. Start there.

This file defines the **shared rules every task spec assumes** — read it before running any spec.

> Scope note: `tournament-server/` is **not** a maichess service (see the root `CLAUDE.md`).
> Never modify it. Its `api/openapi.yaml` is consumed only as an external contract — originally by
> the aborted external-games prompt, now by
> [16-lichess-bridge-compatibility](planned/16-lichess-bridge-compatibility.md).

### Architecture ADRs for the second batch (read before the relevant prompt)

- [caching-and-read-models.md](../knowledge/architecture/caching-and-read-models.md) — CQRS read
  models, Redis-as-default cache, the Redis-vs-KTable split. (`09`, `11`, `12`)
- [change-data-capture.md](../knowledge/architecture/change-data-capture.md) — Debezium CDC; the
  rule "event-sourcing for match, CDC/outbox for CRUD stores, never both." (`10`, `13`)
- [search-service.md](../knowledge/services/search-service.md) — ES as a derived,
  rebuildable read model. (`13`)
- [anticheat-service.md](../knowledge/services/anticheat-service.md) — detection, storage,
  flag propagation, matchmaking toggle. (`14`)
- [serialization-protobuf-migration.md](../knowledge/architecture/serialization-protobuf-migration.md)
  — Avro→Protobuf dual-serde + registry removal. (`15`)

### New repos / services / infrastructure to pre-create

Tell the user to create these in advance (they are out of scope to scaffold from nothing here):

- **`maichess-search-service`** — new repo (`13`).
- **`maichess-anticheat-service`** — new repo (`14`).
- **`anticheat-db`** — a new **DatabaseService instance** (Mongo), not a new repo (`14`).
- **Infra (deploy-only, no repo):** Kafka Connect + Debezium connectors (`10`, `13`),
  Elasticsearch (`13`), `cheat.events.v1` topic (`14`). Redis already ships and is reused (`09`,
  `11`, `12`).

## Architecture decisions (already made — implement, don't re-litigate)

1. **Bot-setup orchestration lives in a new dedicated ASP.NET microservice**,
   `maichess-bot-arena-service`. It owns setup collections, pairings, best-of-3
   aggregation, FEN-list expansion, color alternation, the global concurrency
   limit, and result storage. It spawns games via the existing match-maker
   **bot-vs-bot** path and reads live state from match-manager.
2. **The dev toggle is a persisted user-service profile field** (`dev_mode`),
   surfaced through proto + REST + DB. Not client-only.
3. **Ratings use true chess.com-style Glicko-2** (rating + deviation + volatility
   per user). Seed/base rating is **400**. The displayed `elo` is the Glicko-2
   rating rounded.
4. **External games: the engine drives, we mirror.** A bridge streams the external
   game, calls the maichess Engine for each move, submits it to the external API,
   and writes a read-only `external` match into match-db so it appears in Watch
   and Past Matches with an "external" tag.

## Non-negotiable project conventions (apply in every prompt)

These come from the root `CLAUDE.md`, the per-repo `CLAUDE.md` files, and the
`maichess-knowledge-base/`. Re-read the relevant ones at the start of each prompt.

1. **Contracts are the source of truth.** Before writing service code, update the
   contract in `maichess-api-contracts/` (proto under `protos/<service>/v1/`,
   REST under `rest/<service>.md`). Follow the protobuf style rules
   in `maichess-api-contracts/CLAUDE.md` (snake_case fields, PascalCase
   messages/services, SCREAMING_SNAKE_CASE enum values, one service per file,
   `v1/` versioning). Never infer a contract from an existing implementation.
2. **Contract versioning is an explicit, prompted handoff.** The contracts repo
   publishes the `Maichess.PlatformProtos` package (NuGet + Scala/TS) on a `v*`
   git tag. Whenever a prompt changes `maichess-api-contracts/`, it MUST:
   - **Stop and instruct the user** to commit, tag (`vX.Y.Z`), and push the
     contracts repo so the package builds and becomes available. Do not attempt
     to consume a version that has not been published.
   - Then **bump the pinned version in every consuming service**: the
     `Maichess.PlatformProtos` `PackageReference` in each `*.csproj`, and the
     Scala Maven coordinate in each `build.sbt`. Reconcile **all** services to the
     newly published version, not just the one you touched (versions have drifted
     between services before).
   - If a service compiles against the new contract but you cannot yet verify
     because the package is unpublished, document the blocker in that service's
     `CONTRACT_NOTES.md` per the Contract Policy and stop.
3. **Tests are a mandatory deliverable.** Every code change ships with tests.
   Target **100% line, branch, and method coverage** on non-excluded code.
   - Apply `[ExcludeFromCodeCoverage]` only to the categories the root `CLAUDE.md`
     lists (REST endpoint adapters, repositories needing live deps, fire-and-forget
     async, `[LoggerMessage]` partials, REST DTO records).
   - Configure coverlet `ExcludeByFile` for `Program.cs`, `*.g.cs`, `*.generated.cs`.
   - Verify with `dotnet test -p:CollectCoverage=true` before considering done.
4. **Mutation testing stays wired and green-enough.** Mirror the new code's
   coverage exclusions into the service's Stryker.NET config. Mutation testing is
   an audit, not a gate, but surviving mutants you introduce should be reviewed.
5. **Match the surrounding code.** Reuse existing patterns and utilities rather
   than inventing new ones — each prompt names the specific reusable pieces.
6. **New services persist via a dedicated database-service.** Any new service that
   stores data MUST go through the generic **database-service** gRPC CRUD contract
   (`protos/database-service/v1/database.proto`), as user-service does in
   `UsersService.cs` (`Database.DatabaseClient`, `Struct` records) — **never a
   direct Mongo/SQL driver.** Provision a dedicated database-service instance for the
   new domain rather than reaching into another service's store. (Match-manager's
   direct `MongoDB.Driver` usage is legacy; do not copy it for new services.)

## Client conventions (prompts 01, 02, 05, 06)

From `maichess-client/CLAUDE.md`:
- **Next.js 16** — read the relevant guide in `node_modules/next/dist/docs/` (or
  use the context7 MCP) before writing Next-specific code. Conventions differ from
  training data.
- Prefer **server components**; push client behaviour into specialized hooks under
  `lib/hooks/`, models into `lib/models/`, helpers into `lib/utils/`.
- Path alias `@/*` resolves to the project root.
- No test framework is configured for the client; verify with `npm run build` and
  `npm run lint`, plus a manual click-through.

## External contracts consumed (prompt 06 only)

- **Lichess Bot API** — https://lichess.org/api#description/clients (board/bot
  stream + move endpoints, NDJSON). Reference only; nothing to generate.
- **tournament-server** — `tournament-server/api/openapi.yaml` (NDJSON tournament
  + game streams, `POST .../move/{uci}`). Read-only external contract.

## Per-prompt checklist (each prompt restates this)

- [ ] Read the relevant contracts + knowledge-base docs first.
- [ ] Update `maichess-api-contracts/` (if applicable) and prompt the user to
      publish a new tagged version, then bump all consumers.
- [ ] Implement the service/client change against the contract.
- [ ] Write/extend tests to 100% coverage; update Stryker config exclusions.
- [ ] Update deployment config where a new service/topology is introduced.
- [ ] Record durable design decisions in `maichess-knowledge-base/`.
- [ ] Run the prompt's verification section end-to-end.
