# maichess Knowledge Base

The home for maichess's durable design knowledge **and** its implementation task specs. Two top-level
areas, deliberately separated:

```
knowledge/        ← how the system IS and WHY (durable; evolves with the design)
tasks/            ← the WORK: specs to build it, and their status (planned ↔ done)
```

## knowledge/ — durable truth about the system

Read these to understand the platform. They are the source of truth for design decisions; code
should conform to them, not the other way around.

- **[overview](knowledge/overview.md)**, **[structure](knowledge/structure.md)** — what maichess is
  and the microservice map. Start here.
- **[architecture/](knowledge/architecture/)** — cross-cutting design ADRs:
  [event-driven-architecture](knowledge/architecture/event-driven-architecture.md) (Kafka),
  [serialization-protobuf-migration](knowledge/architecture/serialization-protobuf-migration.md),
  [caching-and-read-models](knowledge/architecture/caching-and-read-models.md) (CQRS),
  [change-data-capture](knowledge/architecture/change-data-capture.md).
- **[services/](knowledge/services/)** — per-service design:
  analysis, anticheat, bot-arena, search, and move-validator position-history.
- **[domain/](knowledge/domain/)** — chess/product domain: ratings (Glicko-2), match history & stats,
  dev-mode, external games.
- **[operations/](knowledge/operations/)** — [deployment-and-environments](knowledge/operations/deployment-and-environments.md)
  (staging/prod, what's enabled where, runbook).

## tasks/ — the work and its status

- **[ROADMAP.md](tasks/ROADMAP.md)** — the status board: every feature/migration task, its status
  (✅ done / 🟡 in progress / ⬜ planned / 🟥 aborted), and the dependency order. **Start here to see
  where things stand.**
- **[conventions.md](tasks/conventions.md)** — the shared rules every task spec assumes (contract
  policy, the contract-versioning handoff, testing/coverage, persistence-via-database-service, client
  conventions).
- **[implemented/](tasks/implemented/)** — specs whose work has shipped (historical record).
- **[planned/](tasks/planned/)** — specs not yet started (or only partly built).
- **[kafka-migration-status.md](tasks/kafka-migration-status.md)** — live progress tracker for the
  multi-phase event-driven (Kafka) migration, which is too large for a single spec.

## How these relate

A task spec in `tasks/` references the `knowledge/` doc(s) that justify it; when a task ships, its
durable decisions are recorded in `knowledge/` and the spec moves to `tasks/implemented/`. Keep
`knowledge/` evergreen and `tasks/` as the changing to-do/done record.
