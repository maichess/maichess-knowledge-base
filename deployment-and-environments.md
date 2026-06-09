# Deployment & Environments

**Status:** Reference / runbook
**Relates to:** [event-driven-architecture.md](event-driven-architecture.md),
[change-data-capture.md](change-data-capture.md),
[caching-and-read-models.md](caching-and-read-models.md)

## Context

maichess runs on a **single k3s node** (one droplet). Both environments live on that
node as separate namespaces, sharing the chart in `maichess-deploy`. Because the node
is small, a full `kubectl rollout restart` is slow and resource-tight — expect rollouts
to take minutes and avoid restarting everything unless a deploy requires it.

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
- **Mutable nightly tags + `imagePullPolicy: Always`:** a pod restart re-pulls the
  latest `:nightly`. A new image can therefore land against an *unchanged* (older)
  rendered chart — if the chart adds a new required env/secret, the new image will
  crash until the chart is re-applied. Keep chart and images in lockstep.

## Access

- SSH `ionos` → `root` on the node (full access; reads the kubeconfig, edits
  `/etc/maichess/*`).
- SSH `ionos-maichess` → the `maichess` deploy user; the checked-out deploy repo is
  `/home/maichess/maichess-deploy`.
- kubeconfig: `/etc/rancher/k3s/k3s.yaml`.

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
