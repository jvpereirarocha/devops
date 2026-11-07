# How to deploy OpenObserve on the Kubernetes cluster

> **Summary:** run OpenObserve, a lightweight log (and trace/metric) backend, as a
> single-replica StatefulSet in "local" mode on a single-node k3s cluster. Its UI goes behind
> the Traefik Ingress, Prometheus scrapes its metrics, and the setup is checked end to end by
> sending a test log and searching for it. This is the first step toward centralizing the
> application logs.

- **Last updated:** 2026-11-07
- **Components:** k3s v1.34 (bundled Traefik, `local-path` StorageClass), OpenObserve v1.1.2, Prometheus (service discovery by annotations), Homepage
- **Level:** intermediate

## Context

Until now the only way to read an application's logs in this cluster was `kubectl logs`.
That shows only the current container (and the previous one at most), only one pod at a
time, with no search. When a pod restarts, the old logs disappear.

**OpenObserve** stores logs and lets you search them with SQL or full-text queries from a
web UI. It was chosen because it is a single binary that runs comfortably in a few hundred
MiB of RAM. In **local mode** it keeps everything on a local disk: the log data as
compressed Parquet files and its metadata (users, streams, settings) in an embedded SQLite
database. That fits a homelab with one node and no object storage.

This tutorial covers only the backend. Shipping the logs of the applications into it (with
a collector such as Fluent Bit) is a separate step.

## Prerequisites

- A running k3s cluster with its default Traefik ingress controller and `local-path`
  StorageClass, and `kubectl` pointing at it.
- A namespace for observability tools. Here it is `observability`, which already runs
  Prometheus and Grafana: `kubectl create namespace observability` if you do not have one.
- Traefik reachable from the LAN (`kubectl get svc -n kube-system traefik` shows its
  external IP, `<TRAEFIK_IP>`).
- A way to resolve a hostname to `<TRAEFIK_IP>`: a DNS record on your router, or an entry
  in `/etc/hosts` on each machine that opens the UI.
- About 300 MiB of free RAM and a few GB of free disk on the node.
- Optional: Prometheus configured to scrape Services annotated with `prometheus.io/scrape`,
  and [Homepage](https://gethomepage.dev) for a dashboard card.

## Overview

```
browser ──> openobserve.<your-local-domain> ──> Traefik (Ingress) ──┐
                                                                    ├──> Service openobserve (ClusterIP :5080/:5081)
apps / log collectors (inside the cluster) ─────────────────────────┘            │
                                                                         Pod openobserve-0 (StatefulSet)
Prometheus ──> scrapes /metrics on the Service                                   │
                                                                         PVC data-openobserve-0 (local-path)
```

1. Create the Secret with the admin credentials (outside git).
2. Create the StatefulSet.
3. Create the Service.
4. Create the Ingress and the DNS record.
5. Add the Homepage card.
6. Check the whole flow: health, UI, ingestion, search and metrics.

All manifests live in `manifests/observability/open-observe/`.

## Step by step

### 1. Create the admin Secret

A **Secret** is the Kubernetes object for sensitive values. A pod can read it as
environment variables or files. OpenObserve creates its first admin user from two
environment variables, `ZO_ROOT_USER_EMAIL` and `ZO_ROOT_USER_PASSWORD`, so we put them in
a Secret and load it into the container.

The Secret is created with a command and **never written to a file in git**. Secret values
are only base64-encoded, and base64 is not encryption: anyone with the file can read them.

```bash
kubectl create secret generic openobserve-secret -n observability \
  --from-literal=ZO_ROOT_USER_EMAIL=<ADMIN_EMAIL> \
  --from-literal=ZO_ROOT_USER_PASSWORD='<ADMIN_PASSWORD>'
```

The e-mail is just the login name for the UI. The password must have at least 8
characters with upper case, lower case, a digit and a symbol, otherwise OpenObserve
refuses to start. The single quotes stop the shell from interpreting symbols such as `$`
or `!`.

**Why a command and not a `*.example.yaml` file?** If the example Secret sits in the same
folder as the other manifests, `kubectl apply -f <folder>/` also applies the example and
overwrites the real Secret with placeholder values. Creating it with a command avoids that.

**How to check** (lists only the key names, never the values):

```bash
kubectl get secret openobserve-secret -n observability -o jsonpath='{.data}' \
  | python3 -c "import sys,json; print(list(json.load(sys.stdin)))"
```

Expected output: `['ZO_ROOT_USER_EMAIL', 'ZO_ROOT_USER_PASSWORD']`.

### 2. Create the StatefulSet

A **StatefulSet** is like a Deployment, but made for applications that keep state on disk.
Its pods get stable names (`openobserve-0`), and each pod gets its own
**PersistentVolumeClaim** (PVC, a request for a disk) created from `volumeClaimTemplates`.
On an update it removes the old pod **before** it creates the new one, so two copies never
write the same data at the same time (see "Technical decisions").

```yaml
# manifests/observability/open-observe/openobserve-statefulset.yaml (excerpt)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: openobserve
  namespace: observability
spec:
  serviceName: openobserve
  replicas: 1
  selector:
    matchLabels:
      app: openobserve
  template:
    metadata:
      labels:
        app: openobserve
    spec:
      securityContext:
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        runAsNonRoot: true
      containers:
        - name: openobserve
          image: openobserve/openobserve:v1.1.2   # pinned, never "latest"
          ports:
            - { name: http, containerPort: 5080 }  # UI + HTTP ingestion API + /metrics
            - { name: grpc, containerPort: 5081 }  # OTLP gRPC ingestion
          env:
            - { name: ZO_DATA_DIR, value: /data }
            - { name: ZO_LOCAL_MODE, value: "true" }
            - { name: ZO_COMPACT_DATA_RETENTION_DAYS, value: "7" }
            - { name: ZO_MEMORY_CACHE_MAX_SIZE, value: "256" }            # MB
            - { name: ZO_MEMORY_CACHE_DATAFUSION_MAX_SIZE, value: "256" } # MB
            - { name: ZO_TELEMETRY, value: "false" }
          envFrom:
            - secretRef:
                name: openobserve-secret
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: "1",  memory: 1Gi }
          readinessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 5
          volumeMounts:
            - { name: data, mountPath: /data }
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 5Gi
```

What each part does:

- **`securityContext`**: runs the process as an unprivileged user instead of root, so a bug
  in the application cannot act as root inside the container. `fsGroup` makes the mounted
  volume writable by that group.
- **`ZO_COMPACT_DATA_RETENTION_DAYS`**: OpenObserve deletes data older than 7 days. This is
  the **real** disk limit, because `local-path` does not enforce the PVC size (see
  "Technical decisions").
- **`ZO_MEMORY_CACHE_*`**: OpenObserve caches recent data and query results in RAM. Without
  a cap, the cache can grow until the container reaches its memory limit and the kernel
  kills it (`OOMKilled`). 256 + 256 MB stays well below the 1 GiB limit.
- **`requests` / `limits`**: `requests` is what the scheduler reserves for the pod; `limits`
  is the ceiling. Measured usage at idle was about 250 MiB and 40m CPU.
- **Probes**: the **readiness probe** decides when the pod receives traffic; the
  **liveness probe** restarts the container if it stops answering. Both call `/healthz`.
  The liveness probe is more tolerant (longer delay, 5 failures) so that a slow start or a
  heavy query does not cause a restart loop.

```bash
kubectl apply -f manifests/observability/open-observe/openobserve-statefulset.yaml --dry-run=server
kubectl apply -f manifests/observability/open-observe/openobserve-statefulset.yaml
kubectl rollout status statefulset/openobserve -n observability --timeout=180s
```

`--dry-run=server` sends the manifest to the API server for validation without saving it.
It catches errors (invalid fields, missing namespace) before anything changes.

**How to check:**

```bash
kubectl get pods,pvc -n observability -l app=openobserve
kubectl get pvc data-openobserve-0 -n observability
kubectl logs -n observability openobserve-0 --tail=50
```

Expected output: `openobserve-0` is `1/1 Running` with 0 restarts, the PVC
`data-openobserve-0` is `Bound`, and the logs show lines such as `Start ingester init`
with no `ERROR` or `panic`.

### 3. Create the Service

A **Service** gives the pod a stable name and IP inside the cluster. A **ClusterIP**
Service (the default type) is reachable only from inside the cluster. That is enough here:
the applications and log collectors run in the cluster and will send data to
`openobserve.observability.svc:5080`, and people reach the UI through the Ingress.

```yaml
# manifests/observability/open-observe/openobserve-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: openobserve
  namespace: observability
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "5080"
    prometheus.io/path: "/metrics"
spec:
  type: ClusterIP
  selector:
    app: openobserve
  ports:
    - { name: http, port: 5080, targetPort: http }
    - { name: grpc, port: 5081, targetPort: grpc }
```

The `prometheus.io/*` annotations are a convention, not a Kubernetes feature: this
cluster's Prometheus uses service discovery and scrapes only the Services that carry them.
OpenObserve publishes its own metrics (ingestion rate, query latency, storage) at
`/metrics`.

```bash
kubectl apply -f manifests/observability/open-observe/openobserve-service.yaml
```

**How to check:**

```bash
kubectl get svc openobserve -n observability
kubectl get endpoints openobserve -n observability
```

Expected output: a `ClusterIP` Service with ports `5080/TCP,5081/TCP`, and an endpoint
pointing at the pod. An empty endpoint list means the selector does not match the pod
labels, or the pod is not Ready yet.

### 4. Create the Ingress and the DNS record

An **Ingress** is a set of HTTP routing rules ("requests for this hostname go to this
Service"). The **ingress controller** (Traefik, bundled with k3s) reads those rules and
does the routing. One Traefik IP can serve many hostnames, so new web UIs do not need a LAN
IP each.

```yaml
# manifests/observability/open-observe/openobserve-ingress.yaml (excerpt)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openobserve
  namespace: observability
spec:
  ingressClassName: traefik
  rules:
    - host: openobserve.<your-local-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: openobserve
                port:
                  name: http
```

You can list more than one `host` rule, for example one for the router DNS name and one
for a name from `/etc/hosts`.

```bash
kubectl apply -f manifests/observability/open-observe/openobserve-ingress.yaml
```

Then point the hostname at Traefik: add a DNS record `openobserve.<your-local-domain>` →
`<TRAEFIK_IP>` on your router, or add the line `<TRAEFIK_IP> openobserve.<your-local-domain>`
to `/etc/hosts` on each client machine.

**How to check:**

```bash
getent hosts openobserve.<your-local-domain>        # should print <TRAEFIK_IP>
curl -s http://openobserve.<your-local-domain>/healthz
curl -s -o /dev/null -w '%{http_code}\n' http://openobserve.<your-local-domain>/web/
```

Expected output: `{"status":"ok"}` and `200`. Before the DNS record exists, you can test
the Ingress by sending the `Host` header directly to Traefik:
`curl -H 'Host: openobserve.<your-local-domain>' http://<TRAEFIK_IP>/healthz`.

Open `http://openobserve.<your-local-domain>` in a browser and log in with the e-mail and
password from step 1.

### 5. Add the Homepage card

In this cluster the Homepage cards come from the static `services.yaml` inside a ConfigMap
(`manifests/homepage/configmap.yaml`). A **ConfigMap** holds configuration files or values
that a pod mounts. Add an entry next to the other services:

```yaml
        - OpenObserve:
            icon: openobserve.png
            href: http://openobserve.<your-local-domain>
            description: Logs
```

```bash
kubectl diff -f manifests/homepage/configmap.yaml        # review the change
kubectl apply -f manifests/homepage/configmap.yaml
kubectl rollout restart deployment/homepage -n homepage
```

The restart is needed because Homepage mounts its config files with `subPath`. A volume
mounted with `subPath` is a copy taken when the container starts, so Kubernetes does not
update it when the ConfigMap changes.

Do **not** also add `gethomepage.dev/*` annotations to the Ingress. Homepage
auto-discovers Ingresses that carry them, so the card would show up twice.

**How to check:** the "OpenObserve" card appears on the dashboard and the link opens the UI.

### 6. Check the whole flow

A pod in `Running` state does not prove the service works. Send a real log line, search
for it, and confirm Prometheus sees the metrics.

```bash
kubectl port-forward -n observability svc/openobserve 15080:5080 &
U=$(kubectl get secret openobserve-secret -n observability -o jsonpath='{.data.ZO_ROOT_USER_EMAIL}' | base64 -d)
P=$(kubectl get secret openobserve-secret -n observability -o jsonpath='{.data.ZO_ROOT_USER_PASSWORD}' | base64 -d)

# Ingest one JSON log line into the "smoke_test" stream of the "default" organization
curl -s -u "$U:$P" -XPOST http://localhost:15080/api/default/smoke_test/_json \
  -d '[{"level":"info","message":"openobserve smoke test"}]'

# Search the last 10 minutes (timestamps are in microseconds)
NOW=$(($(date +%s%N)/1000)); START=$((NOW-600000000))
curl -s -u "$U:$P" -XPOST http://localhost:15080/api/default/_search \
  -H 'Content-Type: application/json' \
  -d "{\"query\":{\"sql\":\"SELECT * FROM smoke_test\",\"start_time\":$START,\"end_time\":$NOW,\"size\":5}}"

# Clean up the test stream
curl -s -u "$U:$P" -XDELETE http://localhost:15080/api/default/streams/smoke_test
kill %1
```

`kubectl port-forward` opens a temporary tunnel from your machine to the Service, so you
can test the in-cluster address without going through the Ingress. The credentials are
read from the Secret into shell variables, so they never show up on screen.

Expected output: the ingestion returns `"successful":1,"failed":0`, the search returns
`"total":1` with the test message, and the delete returns `"stream deleted"`.

Check that Prometheus scrapes the new target (run the query in the Prometheus UI, or with
curl):

```bash
curl -s -G http://<PROMETHEUS_ADDRESS>/api/v1/query \
  --data-urlencode 'query=up{job="kubernetes-services"}'
```

Expected output: a result for `openobserve.observability.svc:5080` with the value `1`.

## Technical decisions

### StatefulSet instead of Deployment

- **Decision:** run OpenObserve as a StatefulSet with one replica and a `volumeClaimTemplate`.
- **Why:** with a Deployment and its default `RollingUpdate` strategy, Kubernetes starts the
  new pod before it stops the old one. On a single node a `ReadWriteOnce` volume can be
  mounted by both pods at the same time (RWO limits the volume to one *node*, not one
  pod), so two OpenObserve processes would write the same SQLite file and data files and
  could corrupt them. A StatefulSet replaces `openobserve-0` only after the old pod is gone.
- **Alternatives considered:** a Deployment with `strategy: Recreate` also avoids the
  overlap, but the StatefulSet expresses the intent ("this pod owns this disk") and keeps
  a stable pod and PVC name. The official Helm chart targets the distributed mode (several
  components plus object storage and a coordination service), which is far too heavy for a
  homelab.
- **Trade-offs / risks:** there is one replica, so there is a short downtime during each
  update. Logs sent during that window are lost unless the collector buffers and retries
  (Fluent Bit does).

### Retention as the real disk limit

- **Decision:** `ZO_COMPACT_DATA_RETENTION_DAYS=7` and a nominal 5Gi PVC.
- **Why:** the `local-path` provisioner creates a plain directory on the node and **does
  not enforce** the requested size. OpenObserve could fill the node's disk, and a full disk
  breaks every pod on the node. Retention is the control that actually bounds disk usage.
- **Alternatives considered:** a StorageClass with real quotas (Longhorn, LVM-based CSI
  drivers) would enforce the size, but it adds a storage system to run. Object storage
  (MinIO/S3) moves the data off the node but adds another service.
- **Trade-offs / risks:** you cannot search logs older than 7 days. Watch the node's disk
  (`df -h`) and the volume size
  (`du -sh /var/lib/rancher/k3s/storage/*openobserve*` on the node) during the first weeks,
  then tune the retention.

### Ingress instead of a LoadBalancer IP

- **Decision:** a ClusterIP Service plus a Traefik Ingress for the UI.
- **Why:** the applications send logs from inside the cluster, so only humans need to reach
  OpenObserve from the LAN, and an Ingress needs no extra LAN IP. It also follows the same
  pattern as other web UIs in the cluster.
- **Alternatives considered:** `type: LoadBalancer` with MetalLB, as Grafana and Prometheus
  use here. It works, but it spends one LAN IP per service and has no hostname.
- **Trade-offs / risks:** the UI depends on Traefik and on DNS. The traffic is plain HTTP on
  the LAN; add TLS at Traefik before exposing it beyond a trusted network.

### Admin credentials created by command

- **Decision:** create `openobserve-secret` with `kubectl create secret`, never with a file in
  git.
- **Why:** the manifests folder is applied as a whole with `kubectl apply -f <folder>/`, and
  an example Secret file in that folder would overwrite the real one. The repository is also
  public.
- **Trade-offs / risks:** the Secret is not reproducible from git. If the namespace is
  recreated, recreate the Secret before applying the StatefulSet, otherwise the pod stays in
  `CreateContainerConfigError`. Use a separate ingestion user or token for the log
  collectors, not the admin account.

### Raw manifests, not managed by ArgoCD (yet)

- **Decision:** apply the manifests with `kubectl apply`, like the rest of the
  `observability` namespace.
- **Why:** it keeps the namespace consistent. Moving the whole namespace to ArgoCD is a
  separate change.
- **Trade-offs / risks:** nothing reverts manual changes made directly in the cluster.
  Always change the files in `manifests/` first.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `CreateContainerConfigError` | `openobserve-secret` does not exist in the namespace | Create the Secret (step 1); the pod starts on its own |
| Pod restarts right after start, logs complain about the root user password | Password too weak | Recreate the Secret with a stronger password, then `kubectl delete pod openobserve-0 -n observability` |
| Pod `OOMKilled` | Caches larger than the memory limit, or heavy queries | Lower `ZO_MEMORY_CACHE_*` or raise `limits.memory` |
| `404` from Traefik on the hostname | `host` in the Ingress differs from the name in the browser | Make the Ingress rule and the DNS record match exactly |
| Hostname does not resolve | Missing DNS record or `/etc/hosts` entry | Add it (step 4); test meanwhile with `curl -H 'Host: ...'` |
| Homepage card appears twice | Static entry **and** `gethomepage.dev/*` annotations on the Ingress | Keep only one of them |
| Prometheus API answers with an error for `up{job="..."}` | The query was not URL-encoded (`{`, `"`) | Use `curl -G --data-urlencode 'query=...'` |

## How to undo

```bash
kubectl delete -f manifests/observability/open-observe/
kubectl delete secret openobserve-secret -n observability
# The PVC is kept on purpose when a StatefulSet is deleted. Delete it to wipe all the logs:
kubectl delete pvc data-openobserve-0 -n observability
```

Then remove the OpenObserve entry from `manifests/homepage/configmap.yaml`, apply it,
restart Homepage, and remove the DNS record.

## References

- OpenObserve documentation: https://openobserve.ai/docs/
- OpenObserve environment variables: https://openobserve.ai/docs/environment-variables/
- Kubernetes StatefulSets: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- k3s local-path provisioner: https://docs.k3s.io/storage
