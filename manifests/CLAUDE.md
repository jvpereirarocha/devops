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

ArgoCD runs `--insecure` (no TLS); Traefik handles TLS termination at the ingress layer. Admin password login is used; the initial password lives in the `argocd-initial-admin-secret` secret in the cluster, not in this repo.

## Architecture

The cluster is organized into these namespaces:

| Namespace | Purpose |
|---|---|
| `cicd-system` | ArgoCD (Kustomize) + resource quota/limit range |
| `databases` | PostgreSQL StatefulSet (fintracker app DB) |
| `homepage` | [Homepage](https://gethomepage.dev) dashboard |
| `metallb-system` | MetalLB L2 pool `192.168.18.2–192.168.18.6` + `192.168.18.8–192.168.18.200` (`.7` is the node) |
| `observability` | Prometheus + Grafana |
| `github-actions` | Self-hosted GitHub Actions runner |

**MetalLB** provides LoadBalancer IPs from the local LAN range. Services that need a stable LAN IP use `type: LoadBalancer` (e.g., Grafana at `.5`, Prometheus at `.6`). The pool skips `192.168.18.7`, the k3s node IP, so MetalLB never hands it to a Service.

**Prometheus** uses Kubernetes service-discovery (`role: service`) and only scrapes services annotated with `prometheus.io/scrape: "true"`. To expose a new service to Prometheus, add that annotation along with `prometheus.io/port` and optionally `prometheus.io/path`.

**Homepage** cards come from the static `services.yaml` in `homepage/configmap.yaml`. To add a service, add an entry there, apply the ConfigMap and run `kubectl rollout restart deployment/homepage -n homepage` (the config files are mounted with `subPath`, so ConfigMap updates are not picked up without a restart). Homepage auto-discovery only works with `gethomepage.dev/*` annotations on **Ingress** objects, not on Services: the annotations on the Grafana and Prometheus Services have no effect.

**GitHub runner** (`pipeline/github-runner/deployment.yaml`) mounts the host Docker socket and uses a PAT stored in a secret. The secret must be created manually before applying the deployment:
```bash
kubectl create secret generic github-runner-secret \
  -n github-actions \
  --from-literal=GITHUB_PAT=<TOKEN>
```

## Secrets Handling

Real secret values are **not committed**: `database/pg-secrets.yaml` is git-ignored, and only `database/pg-secrets.example.yaml` (with placeholders) is versioned. To recreate it, copy the example, fill in the values, and apply it. Never commit a file with real credentials. The GitHub runner PAT secret is intentionally excluded from the repo and must be created via `kubectl create secret`.
