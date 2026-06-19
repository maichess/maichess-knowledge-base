# 02 — MinIO data lake + Spark Operator

> Read [conventions.md](../../conventions.md),
> [architecture/insights-and-spark](../../../knowledge/architecture/insights-and-spark.md), and
> [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md) first.
> Touches `maichess-deploy/` only. **Gate everything to staging first** (enabled in
> `values-staging.yaml`, off in base `values.yaml`) like search/anticheat.

## Goal

Stand up the object-storage data lake and the Spark execution platform on the cluster, pinned and
resource-capped so insights batch work cannot harm the live platform.

## What already exists (reuse it)

- Helm chart `maichess-deploy/helm/maichess/` with per-component templates, staging gating, the
  `affinityPrimary`-style node-affinity helpers, and post-install Job patterns (e.g.
  `kafka-topics-init`, `search-reindex`).
- The k3s topology + constraints in
  [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md) and
  [deployment-and-environments](../../../knowledge/operations/deployment-and-environments.md):
  `maichess-mega` is the compute home; the hub is off-limits for executors; `flost` is burst-only.
- Container publishing convention: per-repo `.github/workflows/docker-publish.yml` → `ghcr.io/maichess`.

## Implementation

1. **MinIO** — `templates/minio.yaml`: StatefulSet + large PVC **pinned to `maichess-mega`**
   (local-path), buckets `insights-raw` / `insights-parsed` / `insights-agg` (create via a small
   post-install Job, mirroring `kafka-topics-init`). Size the PVC per the storage section of
   [spark-and-minio](../../../knowledge/operations/spark-and-minio.md). Credentials via
   `values-secrets-*` (not in git).
2. **Spark Operator** — add the kubeflow `spark-operator` Helm dependency in `Chart.yaml`, namespaced
   to `maichess`. Configure it to schedule driver/executors **only on `maichess-mega`**
   (nodeSelector/affinity + tolerations), with executor requests/limits and a namespace
   **`ResourceQuota`** capping Spark (≈ ≤12 cores / ~20 Gi) and a **low `PriorityClass`**.
3. **RBAC** — `templates/insights-rbac.yaml`: `ServiceAccount` + `Role`/`RoleBinding` letting the
   insights-service create/read/watch `SparkApplication`/`ScheduledSparkApplication` resources, and a
   `ServiceAccount` for the Spark driver to create executor pods.
4. **Custom Spark image** — in the insights repo: `FROM apache/spark:3.5.x` (Scala 2.13) + the
   `sbt-assembly` jobs jar + the S3A/MinIO + Mongo Spark connector jars; published as
   `ghcr.io/maichess/maichess-insights-spark` via `docker-publish.yml`. (The jar contents land in
   tasks 03/04; this task wires the image + publish workflow.)
5. **`insights-db`** DatabaseService instance — Helm wiring for the dedicated Mongo store from task 01.
6. **Optional `flost` burst** — document/enable dynamic-allocation toleration for the map-only parse
   stage only; keep shuffle local (default off).

## Contract changes

None.

## Tests (mandatory)

- Deploy-only: `helm template`/`helm lint` clean; validate the `SparkApplication` CRD installs and a
  trivial sample job runs on `maichess-mega` within the resource cap. No unit-coverage gate (infra).

## Verify

- On staging: MinIO reachable with the three buckets; Spark Operator running; a sample
  `SparkApplication` schedules on `maichess-mega`, respects the `ResourceQuota`, and the hub's stateful
  pods/ingress show no contention. `insights-db` reachable via its DatabaseService.

## Knowledge base

- Update [operations/spark-and-minio](../../../knowledge/operations/spark-and-minio.md) with the
  as-built resource caps / PVC sizes. Mark task 02 `🟡` in [ROADMAP.md](../../ROADMAP.md).
