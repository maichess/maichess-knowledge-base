# Deployment & Environments

**Status:** Reference / runbook
**Relates to:** [event-driven-architecture.md](../architecture/event-driven-architecture.md),
[change-data-capture.md](../architecture/change-data-capture.md),
[caching-and-read-models.md](../architecture/caching-and-read-models.md),
[k3s-performance-and-threading.md](k3s-performance-and-threading.md)

## Context

maichess runs on a **three-node k3s cluster** joined over a **WireGuard mesh**
(hub-and-spoke; nodes addressed on `10.99.0.0/24`). Both environments live on the
cluster as separate namespaces (`maichess`, `maichess-staging`), sharing the chart in
`maichess-deploy`. Only the hub node is internet-facing; the workers are reachable only
over the overlay, so the public ingress / DNS A-records always point at the hub.

## Cluster topology & scheduling

| Node | wg IP | `maichess/role` | Taint | Size | Purpose |
|---|---|---|---|---|---|
| `ubuntu` | 10.99.0.1 | `primary` | none | 6 CPU / 7.7Gi | control-plane + **public ingress** (`217.160.58.173`); holds all stateful data |
| `vps-771074` (`maichess-mega`) | 10.99.0.3 | `primary` | none | 16 CPU / 31Gi | big internal worker added 2026-06 to spread load |
| `flost` | 10.99.0.2 | `burst` | `maichess/role=burst:NoSchedule` | 4 CPU / 15Gi | burst-overflow only (behind NAT, no public IP) |

- **Networking:** flannel **VXLAN** over the wg node IPs. Each node gets a `/24` from
  the `10.42.0.0/16` pod CIDR; services are `10.43.0.0/16`. wg peers only need each
  other's `/32` (VXLAN encapsulates to the node IP).
- **Scheduling (chart helpers in `_helpers.tpl`):** stateful services
  (`affinityPrimary`) hard-pin to `maichess/role=primary` — i.e. `ubuntu` **or**
  `maichess-mega`; the scheduler favours the emptier/bigger mega. Burst-eligible
  services (`affinityBurstEligible` + `tolerationBurst`) prefer primary but may spill
  onto `flost`. Plain app pods (no affinity) schedule on any untainted node and so
  spread across `ubuntu`/`maichess-mega` but never `flost`.
- **Storage pins stateful to `ubuntu`.** The default `local-path` StorageClass binds
  every PVC to the node that first provisioned it (all on `ubuntu`). So StatefulSets
  (postgres/mongo/redis/kafka/tempo) and PVC-backed pods **cannot be rebalanced** to
  another node without migrating their data — only stateless Deployments move.
- **Rebalancing:** a `kubectl rollout restart deployment -n <ns>` reschedules the
  stateless pods; the scheduler then spreads them across `ubuntu` + `maichess-mega`
  (this is how ubuntu's RAM was brought from ~78% to ~47%).

### apiserver-endpoint quirk (affects worker nodes)

`kubectl get endpoints kubernetes` resolves to the hub's **public IP**
(`217.160.58.173:6443`), which is firewalled from the wg-only nodes. Consequences:

- `kubectl exec` / `logs` / `port-forward` to pods on **agent** nodes (`flost`,
  `maichess-mega`) fail with a 502 (the k3s remotedialer dials the public IP and times
  out). Works fine for pods on `ubuntu`.
- Pods that call the Kubernetes API via the `kubernetes.default` service (e.g.
  **kube-state-metrics**, prometheus's in-cluster service discovery) break on the
  worker nodes. They are pinned to the control-plane node via
  `nodeSelector: node-role.kubernetes.io/control-plane=true` (set on
  `prometheus.server` and `prometheus.kube-state-metrics` in `values.yaml`).
- Plain app pods don't use the API (they reach DBs/services via ClusterIP = kube-proxy
  data plane), so they run anywhere. The clean long-term fix is to set the k3s server
  `advertise-address: 10.99.0.1` (keep the public IP in `tls-san`) and restart the
  server — not yet done.

Because everything stateful is on `ubuntu` (8Gi), a full `kubectl rollout restart` is
still resource-tight on that node — avoid restarting everything unless a deploy needs it.

## Environments

| Env | Namespace | Deploy branch | Image tag | Values |
|---|---|---|---|---|
| prod | `maichess` | `maichess-deploy` `main` | `:main` (service repos `main`) | `values.yaml` + `values-prod.yaml` + server secrets |
| staging | `maichess-staging` | `maichess-deploy` `dev` | `:nightly` (service repos `dev`) | `values.yaml` + `values-staging.yaml` + server secrets |

- **Prod auto-deploys** on push to `maichess-deploy` `main` (`deploy.yml`). Its
  `paths-ignore` only covers `.github/**` — **Helm template/values changes on `main`
  DO trigger a prod deploy.** Treat pushes to deploy `main` as production releases.
- **Staging deploys** via the `update-containers.yml` workflow (manual
  `workflow_dispatch`, environment `staging`). It SSHes to the node, checks out
  `dev`, `helm upgrade`s the `maichess-staging` release, then `rollout restart` +
  `rollout status`.
- **`wipe_data` input (fresh-environment toggle).** `update-containers.yml` has an
  opt-in `wipe_data` boolean that, *after* the chart is deployed, runs
  `helm/scripts/maintenance.sh reset-data --with-kafka --yes` for each selected
  environment — clearing Mongo + Redis + Kafka + Elasticsearch so the new images
  start on a clean slate. It is meant for staging. **Production is interlocked:**
  `wipe_data` only touches the `maichess` namespace when the separate
  `confirm_prod_wipe` input is set to exactly `WIPE-PROD-DATA`; otherwise a
  `prod`/`both` run wipes staging only and logs that prod was skipped.
- **Mutable nightly tags + `imagePullPolicy: Always`:** a pod restart re-pulls the
  latest `:nightly`. A new image can therefore land against an *unchanged* (older)
  rendered chart — if the chart adds a new required env/secret, the new image will
  crash until the chart is re-applied. Keep chart and images in lockstep.

## Access

- SSH `ionos` → `root` on the **hub** node `ubuntu` (full access; reads the kubeconfig,
  edits `/etc/maichess/*`, runs `kubectl`/`helm`).
- SSH `ionos-maichess` → the `maichess` deploy user; the checked-out deploy repo is
  `/home/maichess/maichess-deploy`.
- SSH `maichess-mega` → the `vps-771074` worker node (passwordless sudo).
- `flost` has no SSH alias and is unreachable from the hub (no key) — can't be edited
  remotely.
- kubeconfig: `/etc/rancher/k3s/k3s.yaml` (only on the hub).

## Secrets

`.Values.secrets.*` come from **server-side files not in git**:
`/etc/maichess/values-secrets-prod.yaml` and `/etc/maichess/values-secrets-staging.yaml`
(root-owned). The chart's `values.yaml` carries `CHANGE_ME` placeholders that these
files override. Any new secret a service requires must be added to the server file for
each environment, not only to the chart.

## The CDC / event stack (staging only)

`kafkaConnect.enabled: true` (set in `values-staging.yaml`, off in prod) switches on the
whole change-data-capture and read-model stack:

- **Kafka + Schema Registry**, **Kafka Connect + Debezium** source connectors
  (`user-cdc`, `match-cdc`). A `RegexRouter` SMT rewrites every record to the curated
  topic name, so Debezium output lands on `user.cdc.v1` / `match.cdc.v1` (not the raw
  `<prefix>.<schema>.<table>` name).
- **Mongo becomes a single-member replica set `rs0` with keyfile internal auth** (an
  oplog is required for Debezium). This needs:
  - a real base64 `mongoKeyfile` in the server secrets file (the chart default is a
    placeholder — an invalid keyfile makes `mongod` crashloop with
    *"Unable to acquire security key[s]"*);
  - the `mongo-rs-init` post-upgrade Job hook, which `rs.initiate()`s the set. If
    `mongod` can't start, this hook blocks and the `helm upgrade` fails on
    *"post-upgrade hooks failed: timed out"*.
  - When enabled, `mongo-connection-string` gains `?replicaSet=rs0&authSource=admin`.
    **Mongo-consuming services (`match-db`, `arena-db`) must be restarted** to pick up
    the new string, or they keep using the old standalone string and fail auth
    (`Type: Standalone, State: Disconnected, Servers: []`).
- **Consumers must tolerate a not-yet-created topic.** Debezium creates its topic only
  when it first emits; a consumer that subscribes earlier (and stops the host on error)
  will crashloop until the topic exists. Pre-create CDC topics, or make the consumer
  resilient.
- The match-maker **Streamiz** KTable uses `STREAMIZ_STATE_DIR` (an `emptyDir`,
  rebuildable from the changelog).
- `elasticsearch` / `anticheat` / `search-service` templates exist but are gated OFF in
  staging.

## Rollout strategy (why deploys used to hang on old pods)

App Deployments (`_app-deployment.tpl`) ship a **zero-surge** rolling update —
`strategy.type: RollingUpdate`, `maxSurge: 0`, `maxUnavailable: 1` (overridable
per service via the macro's `strategyType` / `maxSurge` / `maxUnavailable`).

Without this they inherited the Kubernetes default (`maxSurge 25%` /
`maxUnavailable 25%`), which at `replicas: 1` rounds `maxUnavailable` **down to 0**:
a new pod must schedule *before* the old one may terminate. Both deploy workflows
restart **every** app service at once (`kubectl rollout restart -l
maichess.io/container-update=true`), and the resource-tight `ubuntu` node can't fit
the surge — the new pod stays `Pending`, the old pod never terminates, and `kubectl
rollout status` blocks until timeout (the "old pods won't shut down / deploy hangs
until the pipeline times out" symptom). Zero-surge frees the old pod's resources
first, then schedules the new one — accepting a few seconds of per-service downtime
(fine pre-launch). See also the full-restart resource note under *apiserver-endpoint
quirk* above and [k3s-performance-and-threading.md](k3s-performance-and-threading.md).

## Operational data maintenance (cleanup & reset)

Routine data chores — purging the stale engine-analysis cache after the default bot
changed, clearing games stuck at `status: ongoing`, flushing caches, rebuilding the
Elasticsearch indexes, or a full clean-slate wipe — are handled by
**`maichess-deploy/helm/scripts/maintenance.sh`**. Run it on the hub
(`./helm/scripts/maintenance.sh <cmd> [--staging] [--yes]`); it talks to each store
via `kubectl exec` so no credentials hit the command line.

Full subcommand reference and recipes live in
**`maichess-deploy/helm/scripts/MAINTENANCE.md`** — not repeated here. Two pitfalls
worth flagging:

- **ES reindex must not run as a blocking Helm hook.** The chart's `search-reindex`
  Job is a `post-upgrade` hook, so `helm upgrade` waits on it and a large rebuild
  stalls the whole deploy until timeout. The `update-containers.yml` `reindex` input
  therefore does **not** enable that hook — it deploys `--no-hooks` and, after the
  rollout, launches `maintenance.sh rebuild-search` (the **detached Job** form, no
  `--wait`), so the rebuild outlives the pipeline instead of timing it out.
  (Historically the input enabled the blocking hook, which made the toggle unusable.)
  Clean stale data first — fewer source docs means a far faster rebuild. (ES is a
  derived, rebuildable read model — never a source of truth; see
  [search-service.md](../services/search-service.md).)
- **Stuck `ongoing` games** are removed at the read-model layer (Mongo `matches` +
  Redis `match:live:*`); the projector does not resurrect them because its
  `match.events.v1` offsets are already committed. A `reset-data --with-kafka` also
  purges the event log itself.

## Validating chart changes safely

A Go-template parse error aborts `helm upgrade` **before** it records a revision, so the
release silently stays on the previous chart while images keep advancing. To catch this
before deploying, render the chart — and to avoid dirtying the node's checked-out git
tree, copy `helm/` to `/tmp` and `helm template` there:

```
cp -r helm /tmp/helm-check
helm template maichess-staging /tmp/helm-check/maichess -n maichess-staging \
  -f /tmp/helm-check/maichess/values.yaml \
  -f /tmp/helm-check/maichess/values-staging.yaml \
  -f /etc/maichess/values-secrets-staging.yaml > /dev/null
```
