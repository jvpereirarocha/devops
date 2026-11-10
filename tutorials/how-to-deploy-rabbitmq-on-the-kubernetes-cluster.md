# How to deploy RabbitMQ on the Kubernetes cluster

> **Summary:** run a single-node RabbitMQ broker as a StatefulSet in its own `messaging` namespace (with a ResourceQuota and LimitRange), expose its management UI through Traefik, scrape its metrics with Prometheus, ship its logs to OpenObserve with Fluent Bit and add it to the Homepage dashboard. Along the way it explains a pitfall that makes RabbitMQ crash on its **second** boot.

- **Last updated:** 2026-11-10
- **Components:** k3s v1.34, RabbitMQ 4.3.7 (`rabbitmq:4.3.7-management-alpine`), Traefik (k3s default), local-path StorageClass, Prometheus, Fluent Bit, OpenObserve, Homepage
- **Level:** intermediate

## Context

A message broker lets services talk asynchronously: a producer publishes a message to a queue and a consumer processes it later, so slow work (sending notifications, processing imports) does not block the request that triggered it. RabbitMQ is a mature broker that speaks AMQP and ships a web UI.

On a single-node homelab the goal is a broker that is small, survives restarts with its data intact, is reachable by in-cluster apps, and is visible in the existing monitoring (metrics and logs).

## Prerequisites

- A k3s (or any Kubernetes) cluster and `kubectl` access.
- A StorageClass for persistent volumes (k3s ships `local-path`).
- An Ingress controller (k3s ships Traefik) and a DNS name (or `/etc/hosts` entry) pointing at it, e.g. `rabbitmq.<your-local-domain>`.
- Optional: Prometheus with annotation-based service discovery, Fluent Bit shipping to OpenObserve, and Homepage (see the other tutorials in this folder).

## Overview

```
in-cluster apps ──AMQP 5672──▶ Service rabbitmq (ClusterIP) ──▶ Pod rabbitmq-0 ──▶ PVC data-rabbitmq-0
browser ──▶ Traefik Ingress (rabbitmq.<your-local-domain>) ──▶ Service rabbitmq :15672 (management UI)
Prometheus ──▶ Service rabbitmq :15692/metrics
Fluent Bit ──tails /var/log/containers/rabbitmq-*──▶ OpenObserve stream "rabbitmq"
```

Files (all under `manifests/`):

| File | What it holds |
|---|---|
| `messaging/namespace.yaml` | The `messaging` namespace |
| `messaging/messaging-resource-quota.yaml` | Caps on CPU/memory/pods/PVCs for the namespace |
| `messaging/messaging-limit-range.yaml` | Default requests/limits for containers that declare none |
| `messaging/rabbitmq/rabbitmq-configmap.yaml` | Extra `rabbitmq.conf` settings |
| `messaging/rabbitmq/rabbitmq-service.yaml` | Headless Service + client Service |
| `messaging/rabbitmq/rabbitmq-statefulset.yaml` | The broker itself |
| `messaging/rabbitmq/rabbitmq-ingress.yaml` | Management UI through Traefik |

## Step by step

### 1. Inspect the image before writing manifests

Pin a version (never `latest`) and look inside the image: which user it runs as, where its config lives and which plugins are already on. This avoids guessing.

```bash
docker pull rabbitmq:4.3.7-management-alpine
docker run --rm --entrypoint sh rabbitmq:4.3.7-management-alpine -c \
  'id rabbitmq; cat /etc/rabbitmq/enabled_plugins; ls /etc/rabbitmq/conf.d; cat /etc/rabbitmq/conf.d/*'
```

What we learned:

- The `rabbitmq` user is **uid 100 / gid 101** (Alpine image).
- `rabbitmq_management` and `rabbitmq_prometheus` are already enabled: no plugin file needed.
- Config is read from `/etc/rabbitmq/conf.d/`, which contains `10-defaults.conf` with `loopback_users.guest = false`, meaning the default `guest/guest` account can log in **from the network**. We will close that.

**Why the `4.3.x` line and not the newest `4.4.0`:** a `.0` release published days ago has had little real-world exposure; the latest patch of the previous minor is the conservative choice for infrastructure.

### 2. Create the namespace, ResourceQuota and LimitRange

A **ResourceQuota** caps the total resources all pods in a namespace can request, so one service cannot starve a single-node cluster. Once a quota on CPU/memory exists, every container **must** declare requests and limits, or it is rejected. A **LimitRange** fills in defaults for containers that do not (handy for a quick `kubectl run` debug pod).

```yaml
# manifests/messaging/messaging-resource-quota.yaml (excerpt)
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 3Gi
    persistentvolumeclaims: "3"
    pods: "5"
```

```bash
kubectl apply -f manifests/messaging/namespace.yaml \
              -f manifests/messaging/messaging-resource-quota.yaml \
              -f manifests/messaging/messaging-limit-range.yaml
```

**How to check:** `kubectl describe quota -n messaging` lists the limits with `Used` at 0.

### 3. Create the admin credentials (outside git)

A **Secret** holds sensitive values. It is only base64-encoded, not encrypted, so it never goes into a public repository. Create it directly in the cluster:

```bash
kubectl create secret generic rabbitmq-secret -n messaging \
  --from-literal=RABBITMQ_DEFAULT_USER=<ADMIN_USER> \
  --from-literal=RABBITMQ_DEFAULT_PASS=<ADMIN_PASSWORD>
```

Tip: generate a strong value with `openssl rand -base64 24`, and read it back later with
`kubectl get secret rabbitmq-secret -n messaging -o jsonpath='{.data.RABBITMQ_DEFAULT_PASS}' | base64 -d`.

RabbitMQ reads these variables **only on the first boot** (empty data volume). When they are set, the `guest` user is never created. Changing the Secret later does nothing: change the password with `rabbitmqctl change_password <ADMIN_USER> <NEW_PASSWORD>` and then update the Secret so both stay in sync.

### 4. Add the configuration (ConfigMap)

A **ConfigMap** stores non-sensitive configuration that is mounted into the pod as files. We mount one file, `20-homelab.conf`, next to the image's own `10-defaults.conf`. Files in `conf.d` load in name order, so ours wins on any key both set.

```yaml
# manifests/messaging/rabbitmq/rabbitmq-configmap.yaml (excerpt)
data:
  20-homelab.conf: |
    loopback_users.guest = true                          # guest only from localhost
    total_memory_available_override_value = 1073741824   # = the container limit (1Gi)
    vm_memory_high_watermark.relative = 0.6              # block publishers at 60% of it
    disk_free_limit.absolute = 2GB                       # block publishers below 2GB free
    log.console = true
    log.file = false
```

- **Memory:** when RabbitMQ crosses the watermark it stops accepting publishes (back-pressure) instead of growing until the kernel kills it. Stating the limit explicitly avoids any doubt about whether the broker reads the cgroup limit correctly; 60% leaves room for the Erlang VM overhead.
- **Disk:** `local-path` volumes are plain directories on the node and do **not** enforce the PVC size. The disk alarm is what protects the node's root partition.
- **Logs:** stdout only, so the container runtime (and Fluent Bit) gets them.

The file is mounted with `subPath`, which keeps the rest of `conf.d` intact but means **ConfigMap updates are not reloaded**: run `kubectl rollout restart statefulset/rabbitmq -n messaging` after changing it.

### 5. Create the Services

A **Service** gives pods a stable name and IP. We need two:

- `rabbitmq-headless` (`clusterIP: None`): the "governing" Service a StatefulSet needs to give each pod a stable DNS name (`rabbitmq-0.rabbitmq-headless.messaging.svc`). It carries the Erlang distribution ports (4369, 25672), used by CLI tools and future clustering.
- `rabbitmq` (ClusterIP): what clients use. Ports 5672 (AMQP), 15672 (UI) and 15692 (Prometheus metrics). Only this one has the Prometheus annotations, so the broker is scraped once, not twice.

```yaml
# manifests/messaging/rabbitmq/rabbitmq-service.yaml (excerpt)
metadata:
  name: rabbitmq
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "15692"
    prometheus.io/path: "/metrics"
```

In-cluster apps connect to `amqp://<USER>:<PASSWORD>@rabbitmq.messaging.svc.cluster.local:5672/`.

### 6. Create the StatefulSet

A **StatefulSet** is like a Deployment, but each pod gets a stable identity (`rabbitmq-0`) and its own volume from `volumeClaimTemplates`. This matters for RabbitMQ: its node name is `rabbit@<hostname>`, and the data directory is tied to that name. A Deployment would give a random hostname on each restart and the broker would come up **empty**.

Key parts (see the full file in `manifests/messaging/rabbitmq/rabbitmq-statefulset.yaml`):

```yaml
spec:
  serviceName: rabbitmq-headless
  replicas: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60     # time to close connections and flush to disk
      securityContext:
        runAsUser: 100                      # rabbitmq user of the image
        runAsGroup: 101
        fsGroup: 101
        fsGroupChangePolicy: OnRootMismatch # see "Common problems"
        runAsNonRoot: true
      initContainers:
        - name: fix-cookie-permissions
          image: rabbitmq:4.3.7-management-alpine
          command: ["sh", "-c", "[ ! -f /var/lib/rabbitmq/.erlang.cookie ] || chmod 600 /var/lib/rabbitmq/.erlang.cookie"]
          # (resources and volumeMounts omitted)
      containers:
        - name: rabbitmq
          image: rabbitmq:4.3.7-management-alpine
          envFrom:
            - secretRef:
                name: rabbitmq-secret
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits:   { cpu: "1",  memory: 1Gi }
          startupProbe:   { exec: { command: ["rabbitmq-diagnostics", "-q", "ping"] }, periodSeconds: 10, failureThreshold: 18 }
          readinessProbe: { exec: { command: ["rabbitmq-diagnostics", "-q", "check_port_connectivity"] }, periodSeconds: 30, timeoutSeconds: 20 }
          livenessProbe:  { exec: { command: ["rabbitmq-diagnostics", "-q", "ping"] }, periodSeconds: 60, timeoutSeconds: 20 }
          volumeMounts:
            - { name: data,   mountPath: /var/lib/rabbitmq }
            - { name: config, mountPath: /etc/rabbitmq/conf.d/20-homelab.conf, subPath: 20-homelab.conf, readOnly: true }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec: { accessModes: ["ReadWriteOnce"], storageClassName: local-path, resources: { requests: { storage: 5Gi } } }
```

The three probes do different jobs:

- **startupProbe:** gives the first boot (database creation, plugin start) up to 3 minutes before the other probes count.
- **readinessProbe:** the pod receives traffic only when the AMQP listener accepts connections.
- **livenessProbe:** restarts the container only if the Erlang node stops answering. It is deliberately slow and tolerant, because restarting a broker is expensive.

```bash
kubectl apply -f manifests/messaging/rabbitmq/
kubectl rollout status statefulset/rabbitmq -n messaging
```

**How to check:**

```bash
kubectl get pods,pvc -n messaging
# rabbitmq-0 1/1 Running, data-rabbitmq-0 Bound

kubectl exec -n messaging rabbitmq-0 -c rabbitmq -- rabbitmqctl -q list_users
# only <ADMIN_USER> [administrator], no guest

kubectl exec -n messaging rabbitmq-0 -c rabbitmq -- sh -c \
  'rabbitmq-diagnostics -q environment | grep -E "total_memory_available|vm_memory_high_watermark|disk_free_limit|loopback_users"'
# shows the values from 20-homelab.conf
```

**Then test a restart.** This is the step that catches the bug in "Common problems":

```bash
kubectl delete pod -n messaging rabbitmq-0
kubectl wait --for=condition=Ready pod/rabbitmq-0 -n messaging --timeout=180s
kubectl exec -n messaging rabbitmq-0 -c rabbitmq -- rabbitmqctl -q list_users   # the user survived
```

### 7. Expose the management UI through Traefik

An **Ingress** routes HTTP traffic by hostname to a Service. Only the UI goes through it; AMQP stays inside the cluster.

```yaml
# manifests/messaging/rabbitmq/rabbitmq-ingress.yaml (excerpt)
spec:
  ingressClassName: traefik
  rules:
    - host: rabbitmq.<your-local-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rabbitmq
                port:
                  name: management
```

**How to check:**

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://rabbitmq.<your-local-domain>/            # 200
curl -s -u <ADMIN_USER>:<ADMIN_PASSWORD> http://rabbitmq.<your-local-domain>/api/overview # JSON with rabbitmq_version
curl -s -o /dev/null -w '%{http_code}\n' -u guest:guest http://rabbitmq.<your-local-domain>/api/overview  # 401
```

### 8. Metrics in Prometheus

Nothing to change in Prometheus: it discovers Services with `role: service` and keeps the ones annotated `prometheus.io/scrape: "true"` (step 5).

**How to check:** in the Prometheus UI, *Status → Targets*, `http://rabbitmq.messaging.svc:15692/metrics` is `UP`. Or from inside the cluster:

```bash
kubectl run -n messaging curl-tmp --rm -i --restart=Never --image=curlimages/curl:8.11.1 -q -- \
  -s http://rabbitmq.messaging.svc:15692/metrics | grep -c '^rabbitmq_'
# a few hundred metric lines
```

`/metrics` returns aggregated metrics (cheap). `/metrics/per-object` has per-queue detail but is heavier; use it only if you need it.

### 9. Logs in OpenObserve with Fluent Bit

Add one input, one `kubernetes` filter and one output to `manifests/observability/fluent-bit/fluent-bit-configmap.yaml`:

```yaml
# inputs
- name: tail
  tag: kube.rabbitmq.*
  path: /var/log/containers/rabbitmq-*_messaging_rabbitmq-*.log   # <pod>_<namespace>_<container>-<id>.log
  multiline.parser: cri
  db: /var/lib/fluent-bit/tail-rabbitmq.db
  read_from_head: true
  storage.type: filesystem

# filters
- name: kubernetes
  match: kube.rabbitmq.*
  kube_tag_prefix: kube.rabbitmq.var.log.containers.

# outputs
- name: http
  match: kube.rabbitmq.*
  uri: /api/default/rabbitmq/_json
  # host, port, auth, format: same as the other outputs
```

RabbitMQ log lines start with a timestamp (`2026-01-01 10:00:00.123456+00:00 [info] ...`), so the existing multiline parser (which starts a record at a timestamp, and applies to every `kube.*` tag) already joins multi-line Erlang crash reports into one record.

Fluent Bit does not reload its config:

```bash
kubectl apply -f manifests/observability/fluent-bit/fluent-bit-configmap.yaml
kubectl rollout restart daemonset/fluent-bit -n observability
```

**How to check:** `kubectl logs -n observability daemonset/fluent-bit | grep rabbitmq` shows `inotify_fs_add(): ... rabbitmq-0_messaging_...` and `{"code":200,"status":[{"name":"rabbitmq","successful":N,"failed":0}]}`. The `rabbitmq` stream appears in the OpenObserve UI.

### 10. Add the Homepage card

```yaml
# manifests/homepage/configmap.yaml (excerpt, services.yaml)
- RabbitMQ:
    icon: rabbitmq.png
    href: http://rabbitmq.<your-local-domain>
    description: Message broker
```

```bash
kubectl apply -f manifests/homepage/configmap.yaml
kubectl rollout restart deployment/homepage -n homepage   # subPath mounts again: no hot reload
```

## Technical decisions

### Plain StatefulSet instead of the RabbitMQ Cluster Operator

- **Decision:** hand-written StatefulSet with the official image.
- **Why:** one node, one replica. The repo already runs PostgreSQL and OpenObserve the same way, so the pattern is familiar and every line is visible.
- **Alternatives considered:** the official Cluster Operator (CRDs + a controller that manages a `RabbitmqCluster`). It shines for multi-node clusters, upgrades and TLS, but adds a controller and CRDs to maintain for a single pod. Helm (Bitnami chart) was also ruled out: the repo uses no Helm, and the chart hides most of the decisions this tutorial explains.
- **Trade-offs / risks:** upgrades, clustering and backups are manual. If more nodes are added later, the operator becomes the better option.

### A `messaging` namespace named by function

- **Decision:** `messaging`, not `rabbitmq`.
- **Why:** matches `databases` and `observability`; another broker (or a schema registry) could live there later under the same quota.
- **Trade-offs / risks:** none worth noting.

### Only the UI is exposed; AMQP stays in-cluster

- **Decision:** Ingress for port 15672, ClusterIP for 5672.
- **Why:** the producers and consumers run in the cluster. Exposing AMQP on the LAN (MetalLB LoadBalancer) would add attack surface for no current use.
- **Alternatives considered:** a LoadBalancer IP for the UI and AMQP. Easy to add later if a client outside the cluster needs the broker.
- **Trade-offs / risks:** the UI runs over plain HTTP on the LAN, like the other homelab UIs. Use a strong admin password.

### Explicit memory and disk limits in `rabbitmq.conf`

- **Decision:** state the 1Gi limit to RabbitMQ and set a 2GB disk alarm.
- **Why:** without them RabbitMQ could grow until the kernel OOM-kills the pod, or fill the node's disk (local-path does not enforce PVC size).
- **Trade-offs / risks:** if the container limit changes, `total_memory_available_override_value` must change with it.

### Credentials from a Secret created outside git

- **Decision:** `RABBITMQ_DEFAULT_USER`/`RABBITMQ_DEFAULT_PASS` from `rabbitmq-secret`.
- **Why:** the repo is public; the image's `guest` default is not acceptable.
- **Trade-offs / risks:** the values are applied only on first boot; password rotation is a `rabbitmqctl` operation (step 3).

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| First boot works; after any restart the pod goes `CrashLoopBackOff` with `Cookie file /var/lib/rabbitmq/.erlang.cookie must be accessible by owner only` | With `fsGroup` set, the kubelet by default re-applies group permissions (`chmod g+rw`) to **every** file of the volume on each mount. On the first boot the cookie is created after the mount (mode 600); on the second boot it becomes 660 and Erlang refuses to start. | Set `fsGroupChangePolicy: OnRootMismatch` (the kubelet only touches the volume when its root does not match yet) and add the `fix-cookie-permissions` initContainer (`chmod 600` on the cookie) to repair a volume already affected. Always test a pod restart right after the first deploy. |
| Pod `Pending` or rejected with `must specify limits.cpu` / `exceeded quota` | The namespace has a ResourceQuota | Declare requests/limits on every container (including init containers) or raise the quota. |
| Config change has no effect | ConfigMap mounted with `subPath` | `kubectl rollout restart statefulset/rabbitmq -n messaging` |
| Changing `RABBITMQ_DEFAULT_PASS` does not change the password | Only read on first boot | `rabbitmqctl change_password <ADMIN_USER> <NEW_PASSWORD>` |
| Publishers block, UI shows a memory or disk alarm | Watermark or `disk_free_limit` reached | Find the queue growing (no consumer?), free disk, or raise the limits on purpose |
| Broker comes up empty after a change | Hostname/node name changed (e.g. moved to a Deployment or renamed the StatefulSet) | Keep the StatefulSet name; the data is tied to `rabbit@<hostname>` |

## How to undo

```bash
# Homepage card and Fluent Bit pipeline: revert the edits in git, then
kubectl apply -f manifests/homepage/configmap.yaml && kubectl rollout restart deployment/homepage -n homepage
kubectl apply -f manifests/observability/fluent-bit/fluent-bit-configmap.yaml && kubectl rollout restart daemonset/fluent-bit -n observability

# The broker (the PVC survives deleting the StatefulSet)
kubectl delete -f manifests/messaging/rabbitmq/
kubectl delete pvc data-rabbitmq-0 -n messaging     # wipes the data
kubectl delete namespace messaging                  # removes everything left, including the Secret
```

## References

- RabbitMQ configuration: https://www.rabbitmq.com/docs/configure
- Memory and disk alarms: https://www.rabbitmq.com/docs/alarms
- Production checklist: https://www.rabbitmq.com/docs/production-checklist
- Prometheus plugin: https://www.rabbitmq.com/docs/prometheus
- Kubernetes `fsGroupChangePolicy`: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#configure-volume-permission-and-ownership-change-policy-for-pods
- Official image: https://hub.docker.com/_/rabbitmq
