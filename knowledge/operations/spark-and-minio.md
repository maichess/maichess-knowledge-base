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

## As-built (task 02 — `maichess-deploy`)

The data lake + Spark platform landed in the Helm chart, **staging-gated** (on in
`values-staging.yaml`, off in base `values.yaml`):

- **Node pinning.** Both the hub (`ubuntu`) and `maichess-mega` share
  `maichess/role=primary`, so the role-affinity helper is *not* enough to keep heavy work
  off the hub. MinIO + the Spark Operator pods pin by **hostname** via the
  `maichess.nodeSelectorCompute` helper (`kubernetes.io/hostname`, default `vps-771074`,
  overridable with `insights.computeNodeHostname`). Spark driver/executor pinning is set on
  the `SparkApplication` manifests (tasks 03/04) using the same value.
- **MinIO** (`templates/minio.yaml`): single-node StatefulSet, local-path PVC **100Gi** base
  / **40Gi** staging (raise for full-month ingestion), buckets `insights-raw` /
  `insights-parsed` / `insights-agg` created by a `minio-buckets-init` post-install Job
  (mirrors `kafka-topics-init`). Root creds via `maichess-app-secrets`
  (`minio-root-user` / `minio-root-password`).
- **Spark Operator**: Kubeflow `spark-operator` **2.1.1** Helm dependency (alias
  `sparkOperator`, `fullnameOverride: spark-operator` to keep names RFC1123-lowercase),
  `spark.jobNamespaces` = the release namespace, operator-managed `spark` driver SA.
- **Blast-radius caps** (`templates/spark-resource-governance.yaml`, gated on
  `sparkOperator.enabled`): a low **PriorityClass** `insights-low-priority` (value **-10**,
  preempted first) + a namespace **ResourceQuota** *scoped to that class* at **12 cores /
  20Gi** (requests=limits) — so it caps only Spark, not the stateless replicas sharing mega.
- **RBAC** (`templates/insights-rbac.yaml`): `insights-service` SA + Role/RoleBinding to
  create/watch `SparkApplication`/`ScheduledSparkApplication` and read pod logs.
- **`insights-db`** (`templates/insights-db.yaml`): a DatabaseService Mongo instance like
  `anticheat-db`, gated on `insights.enabled`. Kept **off** until the database-service
  `insights` migration set exists (tasks 03–05) — an unknown migration domain crash-loops
  the instance, so the master `insights.enabled` gate is decoupled from the operator/MinIO
  infra gates.
- **Custom Spark image** (`maichess-insights-spark` repo): Scala **2.13** built from the
  Apache Spark **3.5.3** `scala2.13` distribution tarball (the official `apache/spark` images
  are 2.12-only) + `hadoop-aws` 3.3.4 / `aws-java-sdk-bundle` 1.12.262 (S3A→MinIO) +
  `mongo-spark-connector_2.13` 10.4.1. Published as
  `ghcr.io/maichess/maichess-insights-spark` via `docker-publish.yml`.
- **`flost` burst** (opt-in, default off): `insights.spark.flostBurst.enabled` carries a
  toleration for the map-only parse stage onto the burst node; shuffle/reduce stays local on
  mega.
</content>
