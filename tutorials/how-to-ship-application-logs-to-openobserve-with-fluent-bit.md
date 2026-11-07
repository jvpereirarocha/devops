# How to ship application logs to OpenObserve with Fluent Bit

> **Summary:** run Fluent Bit as a DaemonSet that reads the container log files on the node
> and sends the logs of selected applications to OpenObserve, one stream per application.
> Python tracebacks become a single record, health-check noise is dropped, and each record
> carries the pod name and the container image version. The applications are not changed.

- **Last updated:** 2026-11-07
- **Components:** k3s v1.34 (containerd), Fluent Bit v5.1.5, OpenObserve v1.1.2, Prometheus (service discovery by annotations)
- **Level:** intermediate

## Context

OpenObserve is already running in the `observability` namespace (see
[How to deploy OpenObserve on the Kubernetes cluster](how-to-deploy-openobserve-on-the-kubernetes-cluster.md)),
but nothing sends logs to it yet. This tutorial connects two Python applications in the
`fintracker-dev` namespace: a FastAPI API (`fintracker-deployment`) and a Telegram bot
(`fintracker-bot-deployment`).

**How container logs work in Kubernetes.** An application writes its logs to stdout and
stderr. The container runtime (containerd in k3s) saves them to a file on the node, under
`/var/log/pods/`, and keeps symlinks with readable names in `/var/log/containers/`:

```
/var/log/containers/<pod-name>_<namespace>_<container-name>-<container-id>.log
```

Each line is in **CRI format**: a timestamp, the stream, a flag (`F` for a full line, `P`
for a partial line split by the runtime) and the text:

```
2026-11-07T10:00:00.123456789Z stdout F INFO:     <CLIENT_IP>:1234 - "GET /docs HTTP/1.1" 200 OK
```

`kubectl logs` reads those same files. A **log collector** reads them too and forwards the
lines somewhere else. That is the job of **Fluent Bit**.

## Prerequisites

- OpenObserve running in the cluster and reachable at
  `openobserve.observability.svc:5080`, with its admin credentials in the
  `openobserve-secret` Secret.
- Applications that log to stdout/stderr.
- `kubectl` access, and optionally Docker on your machine to test the config before
  deploying it.
- Optional: Prometheus configured to scrape Services annotated with `prometheus.io/scrape`.

## Overview

```
node: /var/log/containers/<api pods>   ──tail──> multiline ──> drop GET /metrics ──> k8s metadata ──> http ──> stream fintracker_api
node: /var/log/containers/<bot pod>    ──tail──> multiline ───────────────────────> k8s metadata ──> http ──> stream fintracker_bot
                                       (Fluent Bit DaemonSet pod)                                       (OpenObserve)
```

1. Look at the logs you want to ship.
2. Create an OpenObserve service account for Fluent Bit and store its token in a Secret.
3. Give Fluent Bit read-only access to pod metadata (RBAC).
4. Write the Fluent Bit pipeline (ConfigMap) and test it locally.
5. Deploy the DaemonSet and the metrics Service.
6. Check the logs in OpenObserve and the metrics in Prometheus.

All manifests live in `manifests/observability/fluent-bit/`.

## Step by step

### 1. Look at the logs first

Before writing a collector config, know what the logs look like and how much of them is
noise.

```bash
kubectl logs -n fintracker-dev deploy/fintracker-deployment --tail=20
kubectl logs -n fintracker-dev deploy/fintracker-bot-deployment --tail=50
kubectl logs -n fintracker-dev deploy/fintracker-deployment --since=1h | grep -c 'GET /metrics'
```

In this cluster this showed three things that shaped the config:

- The logs are **plain text, not JSON**: uvicorn access logs (`INFO:     ... "GET /docs" 200`)
  and Python `logging` lines (`2026-11-06 22:30:27,439 [ERROR] telegram.ext.Application: ...`).
- Errors come with **multi-line Python tracebacks**. Shipped line by line, one error
  becomes 30 unrelated records.
- In a quiet hour, **all** of the API's lines were Prometheus scraping `/metrics` every 10
  seconds. Shipping them would fill the disk with noise.

### 2. Create the OpenObserve service account and the Secret

A **service account** in OpenObserve is a non-human user with an API token. Fluent Bit uses
it instead of the admin account: the admin password is not copied anywhere else, and the
token can be rotated or revoked on its own.

Create it in the UI (**IAM → Service Accounts → Add**) and copy the token, or use the API.
The token appears only once, in the response to the creation call, so send it straight
into the Secret:

```bash
kubectl port-forward -n observability svc/openobserve 15080:5080 &
U=$(kubectl get secret openobserve-secret -n observability -o jsonpath='{.data.ZO_ROOT_USER_EMAIL}' | base64 -d)
P=$(kubectl get secret openobserve-secret -n observability -o jsonpath='{.data.ZO_ROOT_USER_PASSWORD}' | base64 -d)
SA=<SERVICE_ACCOUNT_EMAIL>      # any e-mail-shaped name, e.g. fluent-bit@<your-local-domain>

T=$(curl -s -u "$U:$P" -XPOST http://localhost:15080/api/default/service_accounts \
      -H 'Content-Type: application/json' \
      -d "{\"email\":\"$SA\",\"first_name\":\"fluent-bit\",\"last_name\":\"collector\"}" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

kubectl create secret generic fluent-bit-openobserve -n observability \
  --from-literal=OPENOBSERVE_USER="$SA" \
  --from-literal=OPENOBSERVE_TOKEN="$T"
```

If you lose the token, generate a new one (the old one stops working):

```bash
curl -s -u "$U:$P" -XPUT "http://localhost:15080/api/default/service_accounts/$SA?rotateToken=true" \
  -H 'Content-Type: application/json' -d '{"first_name":"fluent-bit","last_name":"collector"}'
```

**How to check:** ingest one record with the service account, then delete the test stream:

```bash
curl -s -u "$SA:$T" -XPOST http://localhost:15080/api/default/sa_auth_test/_json -d '[{"m":"x"}]'
curl -s -u "$U:$P" -XDELETE http://localhost:15080/api/default/streams/sa_auth_test
kill %1
```

Expected output: `"successful":1,"failed":0` and then `"stream deleted"`.

### 3. RBAC for pod metadata

Fluent Bit's `kubernetes` filter asks the Kubernetes API for details about the pod that
wrote each line: labels, pod IP, container image. Pods in Kubernetes authenticate to the API
with a **ServiceAccount** (a Kubernetes identity, not the OpenObserve one from step 2). A
**ClusterRole** lists what is allowed, and a **ClusterRoleBinding** grants it to the
ServiceAccount. Here the grant is read-only on pods and namespaces, nothing else.

```yaml
# manifests/observability/fluent-bit/fluent-bit-rbac.yaml (excerpt)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-read
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "list", "watch"]
```

It is a *Cluster*Role because the pods being read live in another namespace than Fluent Bit.

### 4. Write the pipeline and test it locally

Fluent Bit is configured as a **pipeline**: **inputs** read data, **filters** change or drop
records, **outputs** send them. Each record carries a **tag**, and every filter and output
has a `match` pattern that decides which tags it handles. The config is stored in a
ConfigMap (`manifests/observability/fluent-bit/fluent-bit-configmap.yaml`) and uses Fluent
Bit's YAML format.

**Inputs: one `tail` per application.**

```yaml
- name: tail
  tag: kube.api.*
  path: /var/log/containers/fintracker-deployment-*_fintracker-dev_*.log
  multiline.parser: cri
  db: /var/lib/fluent-bit/tail-api.db
  read_from_head: true
  storage.type: filesystem
```

- `path` is a glob on the file name `<pod>_<namespace>_<container>-<id>.log`. Pod names
  start with the Deployment name, so the glob selects one application. Be careful with
  prefixes: `fintracker-*` would also match the bot and the frontend.
- The `*` in `tag` is replaced by the file path, so the tag becomes
  `kube.api.var.log.containers.<file name>`. The `kubernetes` filter reads the pod name out
  of that tag later.
- `multiline.parser: cri` decodes the CRI format and joins the partial (`P`) lines.
- `db` is a small SQLite file where Fluent Bit records how far it has read each file
  (the **offset**). After a restart it continues from there: no duplicates, no gaps.
- `storage.type: filesystem` buffers records on disk instead of only in memory, so a
  backlog survives a restart and does not use up RAM while OpenObserve is down.

**Filter 1: join Python tracebacks.** A **multiline parser** is a small state machine. A
line that matches `start_state` begins a new record; lines that match `cont` are appended
to it:

```yaml
multiline_parsers:
  - name: python_app
    type: regex
    flush_timeout: 1000
    rules:
      - state: start_state
        regex: '/^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}|[A-Z]+:\s)/'
        next_state: cont
      - state: cont
        regex: '/^(?!\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}|[A-Z]+:\s)/'
        next_state: cont
```

A record starts with a `logging` timestamp or a uvicorn level prefix (`INFO:`). Every other
line (`Traceback`, `  File ...`, blank lines, `The above exception ...`, the exception
itself) is appended to the previous record. `(?!...)` is a *negative lookahead*: "the line
does **not** start with...". `flush_timeout` sends the record after 1 second without new
lines. Adapt both regexes to your own log format.

```yaml
- name: multiline
  match: kube.*
  multiline.key_content: log
  multiline.parser: python_app
  emitter_storage.type: filesystem
```

Put the multiline filter first, so that the later filters see whole records.

**Filter 2: drop the noise.**

```yaml
- name: grep
  match: kube.api.*
  exclude: log "GET /metrics HTTP/1\.1"
```

**Filter 3: add Kubernetes metadata.** One instance per tag prefix, because the filter has
to know which part of the tag is the file name:

```yaml
- name: kubernetes
  match: kube.api.*
  kube_tag_prefix: kube.api.var.log.containers.
  buffer_size: 0       # no limit on the API response size (big pods fail with the default)
  labels: on
  annotations: off
```

**Output: one OpenObserve stream per application.**

```yaml
- name: http
  match: kube.api.*
  host: openobserve.observability.svc.cluster.local
  port: 5080
  uri: /api/default/fintracker_api/_json    # /api/<organization>/<stream>/_json
  format: json
  json_date_key: _timestamp
  json_date_format: iso8601
  http_user: ${OPENOBSERVE_USER}             # read from environment variables (step 5)
  http_passwd: ${OPENOBSERVE_TOKEN}
  compress: gzip
  retry_limit: 10
  storage.total_limit_size: 256M             # cap on the disk buffer for this output
```

OpenObserve creates the stream on the first write. `_timestamp` is the field OpenObserve
uses as the event time; here it is the time the runtime saw the line, not the time it was
shipped.

**Test before deploying.** Fluent Bit has a `--dry-run` flag that validates the config, and
you can feed it a sample of real logs. Extract the config from the ConfigMap and run it in a
throwaway container:

```bash
mkdir -p /tmp/fb && cd /tmp/fb
kubectl create -f ~/devops/manifests/observability/fluent-bit/fluent-bit-configmap.yaml \
  --dry-run=client -o jsonpath='{.data.fluent-bit\.yaml}' > fluent-bit.yaml
docker run --rm -e OPENOBSERVE_USER=x -e OPENOBSERVE_TOKEN=y -v /tmp/fb:/cfg:ro \
  fluent/fluent-bit:5.1.5 -c /cfg/fluent-bit.yaml --dry-run
```

Expected output: `configuration test is successful`.

To test the multiline parser, turn `kubectl logs` output into CRI lines and run a small
pipeline that prints to stdout:

```bash
kubectl logs -n fintracker-dev deploy/fintracker-bot-deployment --tail=60 \
  | awk '{print "2026-01-01T00:00:00.0Z stderr F " $0}' > /tmp/fb/bot.log
cat > /tmp/fb/test.yaml <<'EOF'
multiline_parsers:
  # copy the python_app parser from the ConfigMap here
pipeline:
  inputs:
    - {name: tail, tag: t.bot, path: /cfg/bot.log, multiline.parser: cri, read_from_head: true, exit_on_eof: true}
  filters:
    - {name: multiline, match: "t.*", multiline.key_content: log, multiline.parser: python_app}
  outputs:
    - {name: stdout, match: "*", format: json_lines}
EOF
docker run --rm -v /tmp/fb:/cfg:ro fluent/fluent-bit:5.1.5 -c /cfg/test.yaml
```

Expected output: each error is one JSON record whose `log` field holds the `[ERROR]` line,
the whole traceback and the exception line. (If the sample starts in the middle of a
traceback, those first lines come out one by one: there was no start line to attach them to.)

### 5. Deploy the DaemonSet and the Service

A **DaemonSet** runs one copy of a pod on every node. That is what a log collector needs:
each node has its own log files, and a new node gets a collector automatically. On a
single-node cluster it is one pod.

```yaml
# manifests/observability/fluent-bit/fluent-bit-daemonset.yaml (excerpt)
spec:
  template:
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:5.1.5
          args: ["-c", "/fluent-bit/etc/fluent-bit.yaml"]
          envFrom:
            - secretRef:
                name: fluent-bit-openobserve
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          resources:
            requests: { cpu: 20m,  memory: 32Mi }
            limits:   { cpu: 200m, memory: 128Mi }
          readinessProbe:
            httpGet: { path: /api/v1/health, port: http }
          livenessProbe:
            httpGet: { path: /, port: http }
          volumeMounts:
            - { name: config, mountPath: /fluent-bit/etc/ }
            - { name: varlog, mountPath: /var/log, readOnly: true }
            - { name: state,  mountPath: /var/lib/fluent-bit }
      volumes:
        - name: config
          configMap: { name: fluent-bit-config }
        - name: varlog
          hostPath: { path: /var/log }
        - name: state
          hostPath: { path: /var/lib/fluent-bit, type: DirectoryOrCreate }
```

- A **hostPath** volume mounts a directory of the node into the pod. `/var/log` is mounted
  read-only (Fluent Bit only reads it). `/var/lib/fluent-bit` holds the offsets and the disk
  buffer; on the node, unlike an `emptyDir`, it survives when the pod is recreated.
- `envFrom` loads the Secret keys as environment variables, which the config reads as
  `${OPENOBSERVE_USER}` and `${OPENOBSERVE_TOKEN}`.
- The container runs as root because the files under `/var/log/pods` are readable only by
  root. The rest is locked down: no Linux capabilities, read-only root filesystem, no
  privilege escalation.

The Service exposes Fluent Bit's own metrics to Prometheus:

```yaml
# manifests/observability/fluent-bit/fluent-bit-service.yaml (excerpt)
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "2020"
    prometheus.io/path: "/api/v2/metrics/prometheus"
```

```bash
kubectl apply -f manifests/observability/fluent-bit/ --dry-run=server
kubectl apply -f manifests/observability/fluent-bit/
kubectl rollout status daemonset/fluent-bit -n observability
```

**How to check:**

```bash
kubectl get pods -n observability -l app=fluent-bit
kubectl logs -n observability ds/fluent-bit --tail=40
```

Expected output: the pod is `1/1 Running`. The logs show `connectivity OK` for each
`kubernetes` filter, one `inotify_fs_add()` line per log file found, and then
`HTTP status=200` lines from the outputs. The body after each `HTTP status=200` reports
`successful` and `failed` counts: read it, because OpenObserve answers `200` even when it
rejects records (see "Common problems").

### 6. Check the result

Send some traffic to the application, then search in the OpenObserve UI (**Logs** → pick the
stream) or with the API:

```bash
kubectl port-forward -n observability svc/openobserve 15080:5080 &
NOW=$(($(date +%s%N)/1000)); START=$((NOW-3600000000))     # last hour, in microseconds
curl -s -u "$U:$P" -XPOST http://localhost:15080/api/default/_search \
  -H 'Content-Type: application/json' \
  -d "{\"query\":{\"sql\":\"SELECT _timestamp, kubernetes_pod_name, kubernetes_container_image, log FROM fintracker_api ORDER BY _timestamp DESC\",\"start_time\":$START,\"end_time\":$NOW,\"size\":10}}"
```

Expected output: recent requests with the pod name and the image, e.g.
`<your-image>:1.2.0`. OpenObserve flattens nested fields, so the Kubernetes metadata
appears as `kubernetes_pod_name`, `kubernetes_labels_app`, `kubernetes_container_image`,
and so on.

Confirm the noise is gone:

```sql
SELECT count(*) FROM fintracker_api WHERE log LIKE '%GET /metrics%'
```

Expected output: `0`.

Useful searches in the UI:

```sql
-- errors of the bot, newest first
SELECT _timestamp, log FROM fintracker_bot WHERE log LIKE '%[ERROR]%' ORDER BY _timestamp DESC
-- which image version produced the 5xx responses
SELECT kubernetes_container_image, count(*) FROM fintracker_api WHERE log LIKE '%" 5__ %' GROUP BY 1
```

In Prometheus, the query `up{job="kubernetes-services"}` shows the target
`fluent-bit.observability.svc:2020` with value `1`, and
`sum by (name) (fluentbit_output_errors_total)` should stay at `0`. An alert on that metric
tells you when shipping breaks.

## Technical decisions

### A node-level collector instead of changing the applications

- **Decision:** applications keep logging to stdout; Fluent Bit reads the files on the node.
- **Why:** the applications stay unaware of the log backend. If OpenObserve is down, they do
  not slow down or fail, and Fluent Bit buffers and retries. Adding another application is a
  config change in one place, with no redeploy of the app.
- **Alternatives considered:** a logging handler inside each app that posts to OpenObserve
  (OTLP or HTTP). It couples every app to the backend and needs code changes and deploys in
  each repository.
- **Trade-offs / risks:** the collector needs root and a hostPath to `/var/log`. Plain-text
  logs need parsing; structured (JSON) logs in the apps would make fields such as level and
  logger searchable directly.

### Fluent Bit instead of other collectors

- **Decision:** Fluent Bit.
- **Why:** small footprint (about 35 MiB of RAM here), built-in Kubernetes metadata and
  multiline support, on-disk buffering, and OpenObserve documents it as a source.
- **Alternatives considered:** the OpenTelemetry Collector (more general, heavier, more
  verbose config for logs only), Vector (good, but a less common pairing with OpenObserve),
  Promtail (made for Loki and deprecated in favor of Grafana Alloy).
- **Trade-offs / risks:** Fluent Bit does not reload its config on its own; a config change
  needs `kubectl rollout restart daemonset/fluent-bit -n observability`.

### Selecting applications by file name, one stream each

- **Decision:** one `tail` input per application with a path glob, and one OpenObserve
  stream per application.
- **Why:** files of other pods are never even opened, which is cheaper than reading
  everything and filtering afterwards. Separate streams keep searches and the schema of each
  app independent.
- **Alternatives considered:** tail every container in the namespace into a single stream
  and filter by `kubernetes_labels_app` when searching. It is less config, but it also ships
  apps you did not ask for.
- **Trade-offs / risks:** the globs depend on the Deployment names. If a Deployment is
  renamed, its logs silently stop; the missing stream updates in OpenObserve are the hint.

### Dropping health-check logs at the source

- **Decision:** a `grep` filter drops the `GET /metrics` access logs.
- **Why:** they were 100% of the API's volume in a quiet hour and carry no information
  (Prometheus already records whether the scrape worked).
- **Trade-offs / risks:** if you ever need to debug the scrape itself, use `kubectl logs`.

### A service account instead of the admin user

- **Decision:** Fluent Bit authenticates with its own OpenObserve service account.
- **Why:** the admin password is not copied into another Secret, and the token can be
  rotated or revoked without touching anything else.
- **Trade-offs / risks:** the open-source edition of OpenObserve has no role-based access
  control (the `roles` API answers `Not Supported`), so the token is not limited to
  ingestion. Treat it as a privileged credential.

### Not backfilling old logs

- **Decision:** keep OpenObserve's default ingestion window (`ZO_INGEST_ALLOWED_UPTO`,
  5 hours).
- **Why:** with `read_from_head: true` the first run sends what the node still keeps (about
  10 MiB per container), and OpenObserve rejects what is older than 5 hours. Raising the
  window just for a one-off backfill is not worth it; the limit protects the storage from
  late, out-of-order data.
- **Trade-offs / risks:** history from before the deploy is only available with
  `kubectl logs`, and only while the node keeps it.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `CreateContainerConfigError` | The `fluent-bit-openobserve` Secret does not exist | Create it (step 2) |
| Output logs `HTTP status=401` | Wrong user or token | Rotate the token (step 2), recreate the Secret, restart the DaemonSet |
| `HTTP status=200` but `"failed":N` with `Too old data, only last 5 hours data can be ingested` | Backlog older than OpenObserve's ingestion window | Expected on the first run; raise `ZO_INGEST_ALLOWED_UPTO` on OpenObserve only if you need that history |
| No `inotify_fs_add()` for an app | The path glob does not match the file name | `ls /var/log/containers/` on the node and fix the glob |
| A traceback arrives as many records | The start regex does not match your log format, or the multiline filter is not the first filter | Adapt the `start_state` regex and test locally (step 4) |
| `kubernetes` filter logs `could not get meta` / `403` | RBAC missing, or wrong `kube_tag_prefix` | Check the ClusterRoleBinding and that the prefix matches the tag up to the file name |
| Config change has no effect | Fluent Bit does not reload the ConfigMap | `kubectl rollout restart daemonset/fluent-bit -n observability` |
| The stream never appears for an app that rarely logs | Nothing new was logged inside the ingestion window | Wait for a new log line or trigger one |

## How to undo

```bash
kubectl delete -f manifests/observability/fluent-bit/
kubectl delete secret fluent-bit-openobserve -n observability
# On the node: remove the offsets and the disk buffer
sudo rm -rf /var/lib/fluent-bit
```

Then delete the service account in the OpenObserve UI (**IAM → Service Accounts**) and, if
you want, the `fintracker_api` and `fintracker_bot` streams (**Streams**).

## References

- Fluent Bit Kubernetes filter: https://docs.fluentbit.io/manual/pipeline/filters/kubernetes
- Fluent Bit multiline parsing: https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/multiline-parsing
- Fluent Bit YAML configuration: https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/yaml
- OpenObserve log ingestion with Fluent Bit: https://openobserve.ai/docs/ingestion/logs/fluent-bit/
- Kubernetes logging architecture: https://kubernetes.io/docs/concepts/cluster-administration/logging/
