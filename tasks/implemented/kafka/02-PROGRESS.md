# Kafka 02 — Migration progress / handoff notes

> Working log for `02-migrate-live-topics-to-protobuf.md`. Update after each unit of
> work so a fresh agent can continue. Status legend: ✅ done · 🚧 in progress · ⬜ todo.

## Preconditions verified (before starting)

- Task 01 serde helpers **exist and compile**: `ProtobufEventSerdes.cs` (match-manager
  `Events/`, match-maker `Queue/`), `protobuf-serde.ts` (socket-service `src/kafka/`).
- Contracts **v0.6.0** is tagged; `Maichess.PlatformProtos` 0.6.0 is referenced and
  **restores locally** (`dotnet build` of match-maker succeeds in this shell — the
  usual fresh-publish 401 does not apply, the package is already in the NuGet cache).
- Generated proto types live in namespace `Maichess.Events.V1`
  (`OutboundEvent`/`SocketPush`, `MatchmakingEvent`/`PlayerEnqueued`/`PlayersMatched`,
  `MatchCommand`/`CreateMatchCommand`, shared `Player`/`TimeFormat`/`MatchSource`).
- Round-trip tests from task 01 already exist (`ProtobufEventRoundTripTests.cs` in
  both C# test projects; `protobuf-serde.test.ts` in socket-service).

## Key design decisions for this task

1. **Discriminator = Confluent schema-id → registry schema type.** Avro and Protobuf
   share the same Confluent wire framing (magic byte `0` + 4-byte big-endian schema
   id); they can only be told apart by looking up the schema id's `SchemaType` in the
   registry (cached per id). Pure helper `ConfluentFraming.SchemaId(bytes)` extracted
   and unit-tested; the registry lookup lives in the (excluded) consumer glue.
2. **Dual-read is RETAINED as the committed end state** (not deleted after cutover).
   Rationale: the task's *Tests* and *Verify* sections both require each consumer to
   handle **both** an Avro and a Protobuf message, and reversibility is a stated goal.
   Nothing *produces* Avro after this task ("no Avro on the wire" holds), but the Avro
   **read** arm is kept so already-enqueued Avro messages still decode and the cutover
   is reversible. The Avro read arm is removed for good in task **09** (registry
   removal). The Avro read path decodes via the **registry-fetched** writer schema, so
   it does not need the embedded `.avsc` — the `.avsc` files are still deleted (they
   were only used by the Avro *producers*, which are gone). **This is the one
   deliberate deviation from step (c)'s literal "retire the Avro read path"; documented
   here and in the per-service CONTRACT_NOTES.**
3. **WARN on decode failure** added at every consume path (was the root cause of the
   silent socket caveat).
4. **`matchmaking.events.v1` has a second consumer the task body doesn't call out:**
   the Streamiz `UserRatingTopology` (match-maker `Streaming/`) deserializes *every*
   message on the topic before filtering for `PlayerEnqueued`. Nobody produces
   `PlayerEnqueued` today, so the only traffic is `PlayersMatched` (which the topology
   filters out) — but the Avro SerDes would still **throw** on proto bytes. So the
   topology needs genuine dual-read too (NOT just LogAndContinue, which would silently
   drop any future proto `PlayerEnqueued` — exactly the anti-pattern the task warns
   against). Approach: unify the topology's in-memory value type on the proto
   `MatchmakingEvent`, dual-read SerDes maps Avro→proto, `EnqueueReader` works on proto.

## Topic status

### `match.commands.v1` — ✅ DONE
- ✅ Consumer dual-read: match-manager `Events/MatchCommandConsumer.cs` now consumes
  `byte[]`, reads the Confluent schema id (`ConfluentFraming.cs`), looks up the
  registry `SchemaType` (cached `isProtobuf` dict), and routes to the proto reader
  (`MatchCommandReader`) or the Avro reader (`MatchCommandAvroReader`, extracted from
  the old inline code). Both project onto `CreateMatchInput`. WARN logged on
  non-framed messages and on any per-message decode failure (the resilient loop).
- ✅ Producer→proto: `KafkaMatchCreator.cs` builds a `MatchCommand` proto and uses
  `ProtobufEventSerdes.Serializer<MatchCommand>(registry)`. Avro/Reflection removed.
- ✅ Retired all 3 `match.commands.v1.avsc` copies (canonical, match-maker `Kafka/`,
  match-manager `Events/` — the latter was already dead) + both csproj `EmbeddedResource`
  lines. (Consumer Avro arm uses the registry-fetched writer schema, not the file.)
- ✅ Tests: `MatchCommandDualReadTests.cs` (11 tests) — proto mapping (human/bot,
  external, unset created_by, empty id, non-CreateMatch), Avro mapping (built from a
  recovered inline schema), and the `ConfluentFraming` discriminator. Full suites:
  match-manager 262 ✅, match-maker 99 ✅.
- New files: `Events/ConfluentFraming.cs`, `Events/CreateMatchInput.cs`,
  `Events/MatchCommandReader.cs`, `Events/MatchCommandAvroReader.cs` (match-manager).

### `matchmaking.events.v1` — ✅ DONE
- ✅ Producer→proto: `KafkaMatchmakingNotifier.PlayersMatched` builds a proto
  `MatchmakingEvent` via a second `eventsProducer` (proto serde). The class now holds
  TWO producers — `socketProducer` (Avro, for the `matched` socket push) and
  `eventsProducer` (proto). The socket push stays Avro until topic 3.
- ✅ Consumer dual-read: the Streamiz `UserRatingTopology` matchmaking stream now works
  in the proto `MatchmakingEvent` type. New `MatchmakingEventSerDes` (Streamiz
  `ISerDes<MatchmakingEvent>`, excluded glue) discriminates via schema id → registry
  type and maps the Avro arm through pure `MatchmakingAvroToProto.Map`. `EnqueueReader`
  reworked onto `MatchmakingEvent`. `Build` signature: matchmaking serdes is now
  `ISerDes<MatchmakingEvent>`; `BuildDefault` uses `new MatchmakingEventSerDes()`.
  (user.events side untouched — still Avro/GenericRecord; that topic migrates later.)
- ✅ Retired `matchmaking.events.v1.avsc` (canonical + match-maker `Kafka/` embedded) +
  csproj line. (The test `AvroTestData.MatchmakingEventsAvsc` inline schema is kept — it
  builds the Avro GenericRecords that drive the Avro-arm mapper test.)
- ✅ Tests: `MatchmakingAvroToProtoTests.cs` (Avro arm: enqueue/dequeue projection,
  end-to-end through EnqueueReader, no-payload→None, framing). `EnqueueReader` +
  topology tests reworked to feed proto via new `ProtoTestSerDes<T>`/`ProtoTestData`.
  Full match-maker suite: 104 ✅ (was 99).
- New files (match-maker): `Streaming/ConfluentFraming.cs`,
  `Streaming/MatchmakingAvroToProto.cs`, `Streaming/MatchmakingEventSerDes.cs`,
  `Tests/Support/ProtoTestData.cs` (incl. `ProtoTestSerDes<T>`),
  `Tests/Streaming/MatchmakingAvroToProtoTests.cs`.
- **Note for topic 3 / a continuing agent:** `KafkaMatchmakingNotifier` still has the
  Avro `socketProducer` + `socket.outbound.v1.avsc` embedded resource + the
  `LoadSchema`/`NewSocketEnvelope` Avro plumbing. Topic 3 replaces `socketProducer`
  with a proto `OutboundEvent` producer and removes all that.

### `socket.outbound.v1` — ✅ DONE
- ✅ Consumer dual-read: socket-service `src/kafka/consumer.ts` reads the schema id
  (`readSchemaId`, added to `protobuf-serde.ts`), asks the registry whether it's
  `PROTOBUF` (REST `GET /schemas/ids/{id}`, cached), decodes via `decodeOutboundEvent`
  (proto) or `registry.decode` (Avro). Both arms project onto a `NormalizedPush` in the
  new pure `src/kafka/outbound-decode.ts` (`fromProto`/`fromAvro`). WARN on decode fail.
- ✅ Producers→proto: match-manager `KafkaSocketNotifier` emits `OutboundEvent`;
  match-maker `KafkaMatchmakingNotifier.NotifyMatched` emits `OutboundEvent` (its socket
  producer is now proto too — all the Avro `LoadSchema`/`NewSocketEnvelope` plumbing is
  gone).
- ✅ Caveat resolved: no `Socket__Transport: grpc` override existed anywhere in
  `maichess-deploy` (code default is `kafka`; the README-described revert was already
  gone). Added an explicit `Socket__Transport: kafka` to `matchManagerService.env` in
  base `values.yaml` to codify intent. WARN-on-decode logs added on every consume path.
- ✅ Retired `socket.outbound.v1.avsc` (canonical + match-manager `Events/` + match-maker
  `Kafka/`) and both csproj lines (match-maker's EmbeddedResource ItemGroup is now empty
  and removed).
- ✅ Tests: `src/kafka/outbound-decode.test.ts` (9 tests — readSchemaId for proto+Avro
  framing + malformed; fromProto/fromAvro both target variants, union-unwrap, no-payload).
  `npm install && npm run build && npm test` → 12 ✅ (9 new + 3 existing serde).
- New files (socket): `src/kafka/outbound-decode.ts`, `src/kafka/outbound-decode.test.ts`.

## ✅ TASK COMPLETE — all three topics migrated

Final test tallies (all run locally, all green):
- match-manager: **262** · match-maker: **104** · socket-service: **12**.

Docs updated:
- ✅ kafka `README.md`: the 3 topics now listed under Protobuf; caveat removed/resolved.
- ✅ `serialization-protobuf-migration.md`: status block marks task 02 done.
- ✅ CONTRACT_NOTES.md (match-manager, match-maker, socket-service): task-02 section +
  dual-read retention decision + caveat resolution.

Carryover for later tasks (NOT this task):
- Avro **read** arms remain in all three consumers until task `09` removes the registry.
- Remaining `.avsc` (match.events, user.events, analysis.*, cheat.events,
  matchmaking.commands) belong to topics with no producer yet — untouched.
- Staging end-to-end verification (kubectl logs; a live human move appearing without a
  reload with `Socket__Transport: kafka`) could not be run from this shell — left for
  whoever has cluster access. The code path + WARN logs are in place.
