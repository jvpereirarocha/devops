# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **Kubernetes manifests repository** for a homelab cluster running on k3s. It contains raw YAML manifests and Helm values files — there is no build system, test suite, or scripting layer. All resources are applied directly with `kubectl` or installed via `helm`.

## Applying Manifests

Apply a single manifest or directory:
```bash
kubectl apply -f <file-or-directory>/
```

Apply everything recursively:
```bash
kubectl apply -R -f .
```

Dry-run to preview changes without applying:
```bash
kubectl apply -f <file> --dry-run=client
```

Validate YAML syntax:
```bash
kubectl apply -f <file> --dry-run=client --validate=true
```

## ArgoCD (Kustomize)

ArgoCD is installed with Kustomize (no Helm) into `cicd-system`. `ci-cd/kustomization.yaml` pulls the official manifest (pinned version), sets the namespace, adds the namespace/quota/limit range, and patches resources (`argocd-resources.yaml`), server params (`argocd-cmd-params.yaml`) and the ClusterRoleBinding subjects, and adds the Traefik ingress for `argocd.local.lan` (router DNS, used by the homepage) and `argocd.local` (`/etc/hosts`) (`argocd-ingress.yaml`). Step-by-step install instructions are in `../k3s-commands.md` (Phase 2). To install or update:
```bash
kubectl kustomize ci-cd/                                   # preview
kubectl apply -k ci-cd/ --server-side --force-conflicts    # apply (server-side is required: the CRDs are too large for client-side apply)
```

`ci-cd/` must be applied with `-k`, not `-f`: its patch files are partial manifests and are not valid on their own, so exclude it from `kubectl apply -R -f .`.

The ArgoCD `Application` objects live in `ci-cd/applications/` and are applied separately with `kubectl apply -f ci-cd/applications/` (they are not part of the kustomization, so updating ArgoCD never touches them). `fintracker` syncs the API repo (`k8s/`) and owns the `fintracker-dev` Namespace; `front-fintracker` syncs the frontend repo (`k8s/`) into the same namespace without declaring it. Both use automated sync with selfHeal, and the image versions are pinned with `images.newTag` in each repo's `kustomization.yaml`.

ArgoCD runs `--insecure` (no TLS); Traefik handles TLS termination at the ingress layer. Admin password login is used; the initial password lives in the `argocd-initial-admin-secret` secret in the cluster, not in this repo.

## Architecture

The cluster is organized into these namespaces:

| Namespace | Purpose |
|---|---|
| `cicd-system` | ArgoCD (Kustomize) + resource quota/limit range |
| `databases` | PostgreSQL StatefulSet (fintracker app DB) |
| `homepage` | [Homepage](https://gethomepage.dev) dashboard |
| `metallb-system` | MetalLB L2 pool `192.168.18.2–192.168.18.6` + `192.168.18.8–192.168.18.200` (`.7` is the node) |
| `observability` | Prometheus + Grafana + OpenObserve (logs) + Fluent Bit (log collector) |
| `messaging` | RabbitMQ (single-node StatefulSet) + resource quota/limit range |
| `github-actions` | Self-hosted GitHub Actions runner |

**MetalLB** provides LoadBalancer IPs from the local LAN range. Services that need a stable LAN IP use `type: LoadBalancer` (e.g., Grafana at `.5`, Prometheus at `.6`). The pool skips `192.168.18.7`, the k3s node IP, so MetalLB never hands it to a Service.

**Prometheus** uses Kubernetes service-discovery (`role: service`) and only scrapes services annotated with `prometheus.io/scrape: "true"`. To expose a new service to Prometheus, add that annotation along with `prometheus.io/port` and optionally `prometheus.io/path`.

**Homepage** cards come from the static `services.yaml` in `homepage/configmap.yaml`. To add a service, add an entry there, apply the ConfigMap and run `kubectl rollout restart deployment/homepage -n homepage` (the config files are mounted with `subPath`, so ConfigMap updates are not picked up without a restart). Homepage auto-discovery only works with `gethomepage.dev/*` annotations on **Ingress** objects, not on Services: the annotations on the Grafana and Prometheus Services have no effect.

**OpenObserve** (`observability/open-observe/`) is the log backend: a single-replica StatefulSet in local mode (data and SQLite metadata on the `data-openobserve-0` local-path PVC), `openobserve/openobserve` pinned version. In-cluster clients send logs to `openobserve.observability.svc:5080` (HTTP) or `:5081` (OTLP gRPC); the UI is behind the Traefik Ingress at `openobserve.local.lan` / `openobserve.local`. local-path does not enforce PVC size, so `ZO_COMPACT_DATA_RETENTION_DAYS` (7) is the real disk limit. The admin credentials live in the `openobserve-secret` Secret, created manually before applying:
```bash
kubectl create secret generic openobserve-secret -n observability \
  --from-literal=ZO_ROOT_USER_EMAIL=<EMAIL> \
  --from-literal=ZO_ROOT_USER_PASSWORD=<PASSWORD>
```
Deleting the StatefulSet keeps the PVC; delete `data-openobserve-0` explicitly to wipe the data.

**Fluent Bit** (`observability/fluent-bit/`) is a DaemonSet that tails `/var/log/containers` on the node and ships the fintracker API, bot and RabbitMQ logs to OpenObserve streams `fintracker_api`, `fintracker_bot` and `rabbitmq`. The `tail` input globs depend on the workload names (`fintracker-deployment-*`, `fintracker-bot-deployment-*`, `rabbitmq-*` in `messaging`): update them if a Deployment is renamed; add another input/output pair to ship a new app. A custom multiline parser joins Python tracebacks into one record (a line starting with `{` also starts a record), and a `parser` filter turns the API's JSON request lines into columns (`route`, `status`, `duration_ms`, `request_id`...); non-JSON lines pass through unchanged. The API does not log `GET /metrics` (its middleware skips it), so there is no filter for it. Offsets and the on-disk buffer live in the hostPath `/var/lib/fluent-bit`. Fluent Bit does not reload its ConfigMap: run `kubectl rollout restart daemonset/fluent-bit -n observability` after changing it. It authenticates with an OpenObserve service account whose token lives in the `fluent-bit-openobserve` Secret (keys `OPENOBSERVE_USER`, `OPENOBSERVE_TOKEN`), created outside git. OpenObserve rejects records older than 5 hours by default (`ZO_INGEST_ALLOWED_UPTO`), so old backlog is not ingested.

**RabbitMQ** (`messaging/rabbitmq/`) is a single-replica StatefulSet with the official `rabbitmq:<version>-management-alpine` image (pinned), data on the `data-rabbitmq-0` local-path PVC. The stable hostname matters: the node name is `rabbit@rabbitmq-0` and the data is tied to it. In-cluster clients connect to `amqp://rabbitmq.messaging.svc.cluster.local:5672`; AMQP is not exposed outside the cluster. The management UI is behind the Traefik Ingress at `rabbitmq.local.lan` / `rabbitmq.local`, and the `rabbitmq` Service carries the Prometheus annotations (port 15692; the headless Service does not, to avoid a double scrape). Extra settings (guest loopback-only, memory limit hint, `disk_free_limit`) live in `rabbitmq-configmap.yaml`, mounted with `subPath` as `conf.d/20-homelab.conf`, so run `kubectl rollout restart statefulset/rabbitmq -n messaging` after changing it. The admin user comes from the `rabbitmq-secret` Secret (keys `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`), created outside git; it is only read on the first boot (empty volume), so later password changes go through `rabbitmqctl change_password`. Keep `fsGroupChangePolicy: OnRootMismatch` and the `fix-cookie-permissions` initContainer: the default fsGroup policy makes `.erlang.cookie` group-readable on every mount and Erlang then refuses to start. Deleting the StatefulSet keeps the PVC; delete `data-rabbitmq-0` explicitly to wipe the data.

**GitHub runner** (`pipeline/github-runner/deployment.yaml`) mounts the host Docker socket and uses a PAT stored in a secret. The secret must be created manually before applying the deployment:
```bash
kubectl create secret generic github-runner-secret \
  -n github-actions \
  --from-literal=GITHUB_PAT=<TOKEN>
```

## Secrets Handling

Real secret values are **not committed**: `database/pg-secrets.yaml` is git-ignored, and only `database/pg-secrets.example.yaml` (with placeholders) is versioned. To recreate it, copy the example, fill in the values, and apply it. Never commit a file with real credentials. The GitHub runner PAT secret is intentionally excluded from the repo and must be created via `kubectl create secret`.
