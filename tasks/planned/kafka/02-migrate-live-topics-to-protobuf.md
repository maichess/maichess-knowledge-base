# Kafka 02 — Migrate the live topics to Protobuf

> Read the [README](README.md). **Depends on:** `01` (proto schemas + serdes exist).

## Goal

Cut the three topics already carrying Avro traffic over to **Protobuf**, per-topic and reversibly:
`socket.outbound.v1`, `matchmaking.events.v1`, `match.commands.v1`. After this task no Avro is on the
wire and the corresponding `.avsc` files are retired. **Also resolve the open socket caveat** (the
match-manager socket push reverted to gRPC).

## Per-topic procedure (do each topic end-to-end before the next)

For each topic: **(a) consumers read both** Avro and Protobuf (discriminate on the serde magic
byte / registry schema type) → **(b) switch the producer(s)** to Protobuf → **(c) retire** the Avro
read path and delete the topic's `.avsc`. Topic names/keys/partitions are unchanged.

### `match.commands.v1` (smallest — do first as the proving ground)
- Consumer: match-manager `Events/MatchCommandConsumer.cs` → dual-read.
- Producer: match-maker `Queue/KafkaMatchCreator.cs` → switch to the proto serde (the
  `IMatchCreator` seam means only this class changes).
- Retire `match.commands.v1.avsc` (canonical + the match-maker embedded copy).

### `matchmaking.events.v1`
- Producer: match-maker `Queue/KafkaMatchmakingNotifier.cs`. Consumer(s): whatever reads
  `PlayersMatched` (grep). Dual-read → switch → retire `.avsc`.

### `socket.outbound.v1`
- Consumer: socket-service `src/kafka/consumer.ts` → dual-read (ts-proto + Confluent proto deser).
- Producers: match-manager `Events/KafkaSocketNotifier.cs`, match-maker `KafkaMatchmakingNotifier.cs`.
- **Resolve the caveat:** before switching the producer, verify the hop end-to-end in staging
  (`kubectl logs` socket-service + match-manager; grep `outbound|consumer|decode`; distinguish
  consumer-disabled / decode-error / broker-unreachable). Once verified, flip
  `Socket__Transport` back to `kafka` for match-manager in
  `maichess-deploy/helm/maichess/values.yaml` and switch the producer to proto. Retire `.avsc`.

## Notes
- Registry stays (now mediating Protobuf). `BACKWARD` compat preserved.
- The publish/consume paths are fire-and-forget — add a one-line WARN log on decode failure so a
  future silent drop is visible (the root cause of the socket caveat).

## Tests
- Each consumer round-trips **both** an Avro-encoded and a Protobuf-encoded message during the
  dual-read window (then the Avro arm is deleted with the topic's retirement).
- Producer output deserializes correctly downstream (proto).
- C#/Node suites to 100% on non-excluded; serde glue excluded.

## Verify
- During each window: Avro **and** proto messages both consumed correctly.
- After each switch: only proto on the wire for that topic; socket pushes, the `matched` push, and
  match creation all work in staging; ordering/semantics unchanged.
- `socket.outbound` specifically: a live human move appears in real time without a page reload, with
  `Socket__Transport: kafka`.

## Docs to update
- Program [README](README.md) current-state: drop these from "Avro"; remove the socket caveat.
- `serialization-protobuf-migration`: mark these three topics migrated.
