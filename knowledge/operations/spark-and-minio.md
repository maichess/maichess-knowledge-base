# Spark & MinIO — placement, sizing, isolation

> Operational design behind the insights program
> ([architecture/insights-and-spark](../architecture/insights-and-spark.md)). Cross-reference
> [k3s-performance-and-threading](k3s-performance-and-threading.md) for the cluster's CPU/threading
> realities.

## Node placement — single-node by default, not fanned across the mesh

Spark driver + executors are **hard-pinned to `maichess-mega`** (16 CPU / 31 Gi), and MinIO
lives there too (local-path PVCs pin to a node).

- The cluster interconnect is a **WireGuard mesh over the public internet** (flannel VXLAN on
  top). Spark's shuffle-heavy aggregation stages (every opening/endgame/position job) are
  *network-bound*; spreading executors across nodes pushes shuffle over that tunnel and loses
  more to network than the extra cores gain.
- The hub (`ubuntu`, 6c/7.7Gi) hosts **all stateful services** (Postgres, Mongo, Redis, Kafka,
  Elasticsearch) + ingress — it must **never** host executors.
- `flost` (4c/15Gi) is NAT'd, tainted burst-only, with broken apiserver→worker reachability.
- **Opt-in only:** the map-only parse stage (independent per-partition, near-zero shuffle) may
  burst onto `flost` via dynamic allocation + toleration for large corpora; the shuffle/reduce
  stays local. Real scaling means a second beefy worker **in the same datacenter** as
  `maichess-mega` (true LAN), not wider mesh fan-out.

## The `.zst` bottleneck → decompress once, then parse in parallel

A single Lichess `.zst` can only be stream-decoded by one core, which serializes the whole
pipeline if parsed inline. Ingestion therefore: download `.zst` → **decompress once to a
splittable form in MinIO** → parse in parallel across all cores → partitioned Parquet → **delete
the decompressed scratch**. Board replay (for position/endgame FENs) is a *separate parallel
stage* over Parquet, never part of the single-core decode.

## Expected runtime (wide error bars)

Recent Lichess standard month ≈ 90–100M games / ~30 GB `.zst` / ~200 GB uncompressed:

| Phase | Full month | Default (sampled/filtered ~10–15%) |
|---|---|---|
| Download 30 GB `.zst` | ~10–40 min (uplink-bound) | same / reuse |
| Decompress → splittable (1 core, I/O) | ~10–25 min | — |
| Parse → Parquet (16 cores) | ~15–30 min | ~5–10 min |
| Analysis incl. board replay (16 cores) | ~15–45 min | ~5–15 min |
| **End-to-end** | **~1–2 h** | **~15–35 min** |

Dominated by single-core decompress + disk I/O. **Default ingestion applies a
rating/time/sample filter; full-month runs are explicit opt-in.**

## Storage sizing (MinIO PVC on `maichess-mega`)

Transient peak for a full month: ~30 GB raw `.zst` + ~200 GB decompressed scratch + ~40–80 GB
Parquet. Delete the scratch after Parquet is written, keeping only raw + Parquet. Size the PVC
with headroom; ideally give MinIO its own disk.

## Live-platform isolation (blast radius = `maichess-mega` CPU, which we cap)

- **Node-pin** to `maichess-mega` → no contention with the hub's stateful services / ingress.
- **Executor requests/limits + a namespace `ResourceQuota`** cap Spark (e.g. ≤12 cores / ~20 Gi),
  leaving headroom for the stateless replicas that also run on that worker.
- **Low `PriorityClass`** so k8s throttles/evicts Spark before any live service under pressure.
- The monthly `ScheduledSparkApplication` runs **off-peak**.
- **No live-Kafka pressure** — jobs read Parquet from MinIO, not `match.events.v1`.
- **No engine load** — annotations-first; opt-in engine re-analysis is bounded-concurrency +
  off-peak.
- MinIO disk churn is on `maichess-mega`; the latency-sensitive datastores are on the hub, on
  separate disks.

## Observability

Keep Spark metrics + the operator's status flowing into the existing Prometheus/Grafana/Tempo
stack; watch executor CPU/memory and `maichess-mega` headroom (`jvm_threads_current`, node
pressure) during ingestion.
</content>
