# Kubernetes: intermediate concepts

> **In one sentence:** once your app runs and has an address, you need to give it settings and secrets, a place to keep data, a way to know it is healthy, limits on what it can consume and a public front door.

- **Last updated:** 2026-11-11
- **Related:** [hands-on tutorial](../tutorials/how-to-practice-kubernetes-intermediate-concepts.md), [basic concepts](kubernetes-basic-concepts.md), [advanced concepts](kubernetes-advanced-concepts.md), [DNS](dns.md), [tutorial: RabbitMQ](../tutorials/how-to-deploy-rabbitmq-on-the-kubernetes-cluster.md), [tutorial: Fluent Bit logs](../tutorials/how-to-ship-application-logs-to-openobserve-with-fluent-bit.md)

## The idea in plain words

The [basic concepts](kubernetes-basic-concepts.md) got an app running behind a Service. A real app also needs:

1. **Settings** that change per environment (without rebuilding the image).
2. **Passwords** kept out of sight.
3. **Storage** that survives when a Pod is replaced.
4. **Health checks**, so Kubernetes knows when the app is ready and when it is stuck.
5. **Limits**, so one greedy app does not starve the others.
6. **A front door** for web traffic.
7. **Jobs** that run once or on a schedule.

Think of moving from "a cook can cook" to "a kitchen that runs in production": recipes cards (ConfigMaps), a safe (Secrets), a pantry (volumes), a thermometer (probes), a budget (resources) and a reception desk (Ingress).

## ConfigMap

A **ConfigMap** holds non-secret settings as key/value pairs or whole files. The Pod reads them as environment variables or as files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  app.conf: |
    timeout = 30
```

```yaml
# in the Pod spec
containers:
  - name: app
    image: myapp:1.0.0
    envFrom:
      - configMapRef: { name: app-config }     # every key becomes an env var
    volumeMounts:
      - name: cfg
        mountPath: /etc/app                    # every key becomes a file here
volumes:
  - name: cfg
    configMap: { name: app-config }
```

**Why:** the same image can run in dev and prod with different settings.

> **Pitfalls:** env vars are read only when the container *starts*, so changing the ConfigMap needs a Pod restart (`kubectl rollout restart deployment/<name>`). Files mounted as volumes do update on their own after a short delay, **except** when mounted with `subPath` (those never update).

## Secret

A **Secret** is like a ConfigMap, meant for sensitive values: passwords, tokens, certificates. It is used the same way (env var or file).

```bash
kubectl create secret generic db-credentials \
  --from-literal=password='<YOUR_PASSWORD>' -n <namespace>
```

> **Important:** a Secret is only **base64-encoded**, which is *not* encryption: anyone who can read the Secret can decode it. What Secrets give you is separate permissions (you can allow reading ConfigMaps but not Secrets) and the chance to encrypt them at rest. **Never commit real Secrets to git.** Commit a `*.example.yaml` with placeholders, or use a tool such as Sealed Secrets or SOPS (see [advanced concepts](kubernetes-advanced-concepts.md#keeping-secrets-out-of-git)).

## Volumes, PersistentVolume and PersistentVolumeClaim

A container's disk disappears with it. For data that must survive (databases, uploads), Kubernetes uses **persistent storage** split in three pieces:

| Object | What it is | Everyday comparison |
|---|---|---|
| **PersistentVolume (PV)** | A real piece of storage in the cluster | A locker in a storage building |
| **PersistentVolumeClaim (PVC)** | A Pod's *request* for storage ("I need 5 GiB, read/write") | The rental request |
| **StorageClass** | The recipe that creates PVs automatically when a PVC asks | The storage company's price list |

```
Pod ──uses──► PVC "data" ──bound to──► PV (created by the StorageClass)
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]      # one node can mount it read/write
  storageClassName: local-path      # k3s default
  resources:
    requests:
      storage: 5Gi
```

| Access mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read/write by **one node** at a time (most common) |
| `ReadOnlyMany` (ROX) | Read-only by many nodes |
| `ReadWriteMany` (RWX) | Read/write by many nodes (needs special storage such as NFS) |

Other volume kinds you will meet: `emptyDir` (scratch space that dies with the Pod), `configMap` and `secret` (files from those objects), `hostPath` (a folder of the node; avoid it).

> **Pitfall:** with the default `local-path` StorageClass, the data lives **on one node's disk**. It is not replicated and it is not a backup. Back up what matters.

## Probes: how Kubernetes knows the app is healthy

A container being "running" does not mean the app works. **Probes** are small checks Kubernetes runs against it.

| Probe | Question | If it fails |
|---|---|---|
| **startupProbe** | "Has the app finished starting?" | Other probes wait; if it never succeeds, the container is restarted |
| **readinessProbe** | "Can it receive traffic right now?" | The Pod is **removed from the Service** (not restarted) |
| **livenessProbe** | "Is it stuck?" | The container is **restarted** |

```yaml
containers:
  - name: app
    image: myapp:1.0.0
    readinessProbe:
      httpGet: { path: /healthz, port: 8000 }
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      httpGet: { path: /healthz, port: 8000 }
      periodSeconds: 20
      failureThreshold: 3
```

Checks can be an HTTP request (`httpGet`), a TCP connection (`tcpSocket`) or a command (`exec`).

**Why readiness matters:** during a rolling update, the Deployment only moves on after the new Pod is *Ready*. Without a readiness probe, traffic reaches Pods that are still booting.

> **Pitfall:** a liveness probe that is too strict kills a slow-but-healthy app, over and over (restart loop). Be generous with timings.

## Resources: requests and limits

Each container can declare how much CPU and memory it needs and may use.

```yaml
resources:
  requests:            # reserved: used by the scheduler to pick a node
    cpu: 100m          # 100 millicores = 0.1 of a CPU core
    memory: 128Mi
  limits:              # the ceiling
    cpu: 500m
    memory: 256Mi
```

| | Request | Limit |
|---|---|---|
| Meaning | "I need at least this" | "Never more than this" |
| Used by | Scheduler (does the node have room?) | The node's kernel, at run time |
| Exceeding CPU | n/a | The container is **slowed down** (throttled) |
| Exceeding memory | n/a | The container is **killed** (`OOMKilled`) |

Two helpers set defaults and caps per namespace:

- **ResourceQuota:** the *total* a namespace may use (e.g. 4 CPUs, 8 GiB, 20 Pods).
- **LimitRange:** default and maximum values for *each* container when it declares none.

The **QoS class** Kubernetes assigns follows from this: `Guaranteed` (requests = limits), `Burstable` (requests < limits) and `BestEffort` (nothing declared). When the node runs out of memory, `BestEffort` Pods are evicted first.

## Ingress

A LoadBalancer Service uses one IP per app. For web apps there is a better way: **one** entry point that reads the requested hostname and path and routes to the right Service.

Analogy: a building's reception desk. Everyone arrives at the same door, and the receptionist sends them to the right office by the name they ask for.

```
browser ──► <LOADBALANCER_IP> ──► Ingress controller (Traefik)
                                     ├─ app.<your-local-domain>   ──► Service app
                                     └─ api.<your-local-domain>   ──► Service api
```

Two pieces:

- **Ingress:** the *rules* (an object you write).
- **Ingress controller:** the *program that applies them* (Traefik, ingress-nginx...). Without one, Ingress objects do nothing. k3s ships with Traefik.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
spec:
  rules:
    - host: app.<your-local-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port: { number: 80 }
```

The hostname must resolve to the controller's IP (see [DNS](dns.md)). Ingress can also terminate **TLS** (HTTPS) using a certificate stored in a Secret.

## Jobs and CronJobs

| Object | Runs | Example |
|---|---|---|
| **Job** | A Pod until it **finishes successfully** (once) | A database migration |
| **CronJob** | A Job **on a schedule** (cron syntax) | A nightly backup at 03:00 |

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mybackup:1.0.0
```

A Deployment keeps things running forever; a Job runs to completion. Use the right tool.

## StatefulSet and DaemonSet

Two more ways to run Pods, for special cases:

| Object | What it guarantees | Use for |
|---|---|---|
| **Deployment** | Interchangeable Pods with random names | Stateless apps (web, APIs) |
| **StatefulSet** | Stable names (`db-0`, `db-1`), stable storage per Pod, ordered start/stop | Databases, message brokers (RabbitMQ), anything with an identity |
| **DaemonSet** | **One Pod on every node** | Log collectors (Fluent Bit), monitoring agents |

A StatefulSet is usually paired with a **headless Service** (`clusterIP: None`), which returns each Pod's own address instead of a single one, so `db-0.db` always reaches that exact Pod.

## Rolling updates in detail

A Deployment replaces Pods gradually, controlled by:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # may create 1 extra Pod above the desired number
    maxUnavailable: 0    # never go below the desired number
```

With 3 replicas, `maxSurge: 1` and `maxUnavailable: 0`: start 1 new Pod, wait until it is Ready, stop 1 old Pod, repeat. Zero downtime, provided the readiness probe is correct. Use `strategy: Recreate` only when two versions must never run together (it stops all old Pods first, causing downtime).

`revisionHistoryLimit` controls how many old ReplicaSets are kept for rollback; see the [tutorial](../tutorials/how-to-limit-the-old-replicasets-kept-by-a-deployment.md).

## Trying it yourself

The hands-on exercises for this level live in a tutorial: [How to practice Kubernetes intermediate concepts](../tutorials/how-to-practice-kubernetes-intermediate-concepts.md).

## Common confusions

| You might think... | But actually... |
|---|---|
| "Secrets are encrypted." | They are base64-encoded. Protect access, and never put them in git. |
| "I changed the ConfigMap but the app did not notice." | Env vars need a restart; `subPath` mounts never refresh. |
| "Readiness and liveness are the same." | Readiness removes the Pod from traffic; liveness restarts it. |
| "Limits reserve resources." | Requests reserve; limits only cap. |
| "Deleting a Pod deletes its data." | Data in a PVC survives. Deleting the *PVC* is what loses it (depending on the reclaim policy). |
| "Ingress exposes my app." | Ingress is only rules. It needs an Ingress controller, and the name must resolve to it. |
| "A PVC with `local-path` is a backup." | It is one disk on one node. |

## Summary

- **ConfigMap/Secret** separate settings from images; Secrets are encoded, not encrypted.
- **PVC + StorageClass** give Pods data that survives restarts.
- **Probes** tell Kubernetes if a Pod is ready (traffic) or stuck (restart).
- **Requests/limits**, **ResourceQuota** and **LimitRange** keep apps from starving each other; memory over the limit means `OOMKilled`.
- **Ingress** + controller route web traffic by hostname; **Job/CronJob**, **StatefulSet** and **DaemonSet** cover batch, stateful and per-node workloads.

## References

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) and [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
