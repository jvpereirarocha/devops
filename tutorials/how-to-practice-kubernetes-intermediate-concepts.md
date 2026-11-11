# How to practice Kubernetes intermediate concepts

> **Summary:** a hands-on lab in a throwaway namespace covering quotas and default limits, ConfigMaps, Secrets, persistent storage, readiness and liveness probes, an OOMKilled container, an Ingress and a CronJob. Each step makes something happen that you can see.

- **Last updated:** 2026-11-11
- **Components:** any Kubernetes cluster (written for k3s with Traefik and the `local-path` StorageClass), `kubectl`, public images `nginx`, `busybox` and `polinux/stress`
- **Level:** intermediate
- **Theory:** [Kubernetes: intermediate concepts](../concepts/kubernetes-intermediate-concepts.md)
- **Do first:** [How to practice Kubernetes basic concepts](how-to-practice-kubernetes-basic-concepts.md)

## Context

The basics get an app running. This lab covers what a production-minded app needs: limits that protect the cluster, configuration and secrets, data that survives, health checks and a front door.

## Prerequisites

- The basic lab done, or equal knowledge (Deployment, Service, labels).
- `kubectl` access with permission to create Namespaces, ResourceQuotas, PVCs and Ingresses.
- An Ingress controller (k3s ships with Traefik) for step 8.
- A second terminal for `kubectl get ... -w`.

## Overview

1. Namespace with ResourceQuota and LimitRange.
2. ConfigMap as environment variables and as a file.
3. Secret.
4. Persistent storage that survives a Pod deletion.
5. Readiness probe: traffic control.
6. Liveness probe: automatic restart.
7. Memory limit: `OOMKilled`.
8. Ingress.
9. CronJob.
10. Clean up.

## Step by step

### 1. Namespace with ResourceQuota and LimitRange

```bash
kubectl create namespace practice-mid
kubectl config set-context --current --namespace=practice-mid

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    pods: "10"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    persistentvolumeclaims: "3"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 50m, memory: 64Mi }
      default: { cpu: 200m, memory: 128Mi }
EOF
```

**Why:** the **ResourceQuota** caps the *total* the namespace may use. Once it limits CPU/memory, every Pod must declare resources, and the **LimitRange** fills in defaults for containers that do not.

**How to check:**

```bash
kubectl describe resourcequota quota
kubectl describe limitrange defaults
```

Expected: `Used` is all zeros; the defaults are listed. Create a Pod and compare:

```bash
kubectl run probe --image=nginx:1.27
kubectl get pod probe -o jsonpath='{.spec.containers[0].resources}{"\n"}'
kubectl describe resourcequota quota | grep -E 'pods|requests.memory'
kubectl delete pod probe
```

You never wrote `resources`, yet the Pod has them (from the LimitRange) and the quota counted them.

### 2. ConfigMap: environment variables and files

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  GREETING: hello-from-configmap
  index.html: |
    <h1>Served from a ConfigMap</h1>
---
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: web
      image: nginx:1.27
      envFrom:
        - configMapRef: { name: app-config }
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      configMap:
        name: app-config
        items:
          - key: index.html
            path: index.html
EOF
```

**How to check:**

```bash
kubectl exec cfg-demo -- printenv GREETING
kubectl exec cfg-demo -- cat /usr/share/nginx/html/index.html
```

Expected: `hello-from-configmap` and the `<h1>` line. The same ConfigMap fed an env var and a file.

**Experiment (the classic pitfall):**

```bash
kubectl patch configmap app-config --type merge -p '{"data":{"GREETING":"changed","index.html":"<h1>Changed</h1>\n"}}'
sleep 60
kubectl exec cfg-demo -- printenv GREETING                       # still the old value
kubectl exec cfg-demo -- cat /usr/share/nginx/html/index.html    # updated
```

Environment variables are read once at start; mounted files are refreshed by the kubelet after a short delay (about a minute). Files mounted with `subPath` would not refresh either.

### 3. Secret

```bash
kubectl create secret generic db-credentials --from-literal=password='<YOUR_PASSWORD>'
kubectl get secret db-credentials -o jsonpath='{.data.password}{"\n"}' | base64 -d; echo
```

**What you learned:** the value is only base64-encoded; anyone allowed to read the Secret can decode it in one command. Protect *access* to Secrets and never commit them to git.

Use it in a Pod as an environment variable:

```bash
kubectl run secret-demo --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"secret-demo","image":"busybox:1.36","command":["sh","-c","echo password length is ${#DB_PASSWORD}; sleep 3600"],"env":[{"name":"DB_PASSWORD","valueFrom":{"secretKeyRef":{"name":"db-credentials","key":"password"}}}]}]}}'
kubectl logs secret-demo
kubectl delete pod secret-demo cfg-demo
```

### 4. Persistent storage

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
kubectl get pvc data
```

On k3s the `local-path` StorageClass uses `WaitForFirstConsumer`, so the PVC stays `Pending` until a Pod uses it. That is normal. Now use it:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/log.txt; sleep 3600"]
      volumeMounts:
        - { name: data, mountPath: /data }
  volumes:
    - name: data
      persistentVolumeClaim: { claimName: data }
EOF
kubectl wait --for=condition=Ready pod/writer --timeout=60s
kubectl get pvc data                 # now Bound
kubectl exec writer -- cat /data/log.txt
```

**The test:** delete the Pod and create it again.

```bash
kubectl delete pod writer
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/log.txt; sleep 3600"]
      volumeMounts:
        - { name: data, mountPath: /data }
  volumes:
    - name: data
      persistentVolumeClaim: { claimName: data }
EOF
kubectl wait --for=condition=Ready pod/writer --timeout=60s
kubectl exec writer -- cat /data/log.txt
```

**Expected:** **two** lines: the file survived the Pod. Data lives in the PVC, not in the container.

> `local-path` keeps the data on one node's disk. It is not replicated and not a backup.

### 5. Readiness probe: traffic control

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ready-demo
spec:
  replicas: 2
  selector:
    matchLabels: { app: ready-demo }
  template:
    metadata:
      labels: { app: ready-demo }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          readinessProbe:
            httpGet: { path: /ready.html, port: 80 }
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ready-demo
spec:
  selector: { app: ready-demo }
  ports:
    - port: 80
EOF
kubectl get pods -l app=ready-demo
kubectl get endpoints ready-demo
```

**Expected:** Pods `0/1 Ready` and `ENDPOINTS <none>`. The container is running, but `/ready.html` does not exist (404), so the probe fails and the Pod receives no traffic. It is **not** restarted: that is the difference from liveness.

Make one Pod ready:

```bash
POD=$(kubectl get pod -l app=ready-demo -o name | head -1)
kubectl exec $POD -- sh -c 'echo ok > /usr/share/nginx/html/ready.html'
sleep 10
kubectl get pods -l app=ready-demo
kubectl get endpoints ready-demo
```

**Expected:** one Pod `1/1`, and exactly one address in `ENDPOINTS`.

### 6. Liveness probe: automatic restart

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: live-demo
spec:
  containers:
    - name: c
      image: busybox:1.36
      command: ["sh", "-c", "touch /tmp/healthy; sleep 30; rm /tmp/healthy; sleep 600"]
      livenessProbe:
        exec: { command: ["cat", "/tmp/healthy"] }
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 2
EOF
kubectl get pod live-demo -w
```

**Expected:** after ~40 s the `RESTARTS` column becomes `1`, then keeps growing every ~40 s. Stop the watch with Ctrl+C and read why:

```bash
kubectl describe pod live-demo | sed -n '/Events:/,$p'
```

Look for `Liveness probe failed` followed by `Container c failed liveness probe, will be restarted`.

### 7. Memory limit: OOMKilled

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  restartPolicy: Never
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
      resources:
        requests: { memory: 64Mi, cpu: 50m }
        limits:   { memory: 100Mi, cpu: 200m }
EOF
sleep 20
kubectl get pod oom-demo
kubectl describe pod oom-demo | grep -A4 'Last State\|State:'
```

**Expected:** `OOMKilled` with exit code `137`. The program tried to use 200 MiB with a 100 MiB limit, so the kernel killed it. CPU over the limit would only have slowed it down.

### 8. Ingress

```bash
kubectl create deployment web --image=nginx:1.27
kubectl expose deployment web --port=80
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  rules:
    - host: web.<your-local-domain>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port: { number: 80 }
EOF
kubectl get ingress web
```

**How to check** without touching DNS, send the hostname by hand:

```bash
curl -H "Host: web.<your-local-domain>" http://<LOADBALANCER_IP>
curl -i http://<LOADBALANCER_IP>          # no matching host: 404 from the controller
```

**Expected:** the nginx page for the first call and a `404` for the second. The controller chose the Service by the `Host` header. To use the name in a browser, make it resolve to `<LOADBALANCER_IP>` (see [DNS](../concepts/dns.md)).

### 9. CronJob

```bash
kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: c
              image: busybox:1.36
              command: ["sh", "-c", "date; echo hello from a Job"]
EOF
sleep 70
kubectl get cronjob,job,pods
kubectl logs -l job-name --tail=2
```

**Expected:** a Job created each minute, its Pod in `Completed`, and only the last two successful Jobs kept. A Job runs to completion; a Deployment would restart it forever.

### 10. Clean up

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace practice-mid
```

The PVC and its data go with the namespace (the `local-path` reclaim policy is `Delete`).

## Technical decisions

### ResourceQuota plus LimitRange together

- **Decision:** create both in the namespace before anything else.
- **Why:** the quota alone rejects Pods that declare no resources; the LimitRange makes that painless by providing defaults. Together they protect the node from one namespace using everything.
- **Alternatives considered:** only a quota (every manifest must repeat `resources`); neither (a single leaking app can take the whole node).
- **Trade-offs / risks:** defaults can be too small for real apps, causing OOMKilled or throttling; set explicit values for anything important.

### Secrets created on the command line

- **Decision:** `kubectl create secret ... --from-literal` in this lab only.
- **Why:** it shows the mechanism without any file with a password.
- **Alternatives considered:** a YAML Secret in git (never with real values); Sealed Secrets or SOPS for GitOps (see the [advanced concepts](../concepts/kubernetes-advanced-concepts.md#keeping-secrets-out-of-git)).
- **Trade-offs / risks:** the value stays in your shell history; use a placeholder value in a lab.

### Probes with short periods

- **Decision:** 5-second periods and a low `failureThreshold` for the demos.
- **Why:** you see the result in seconds.
- **Alternatives considered:** the defaults (10 s period, threshold 3) are safer for real apps.
- **Trade-offs / risks:** aggressive liveness probes can restart slow-but-healthy apps in a loop. Be generous in production.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| `Error from server (Forbidden): ... must specify limits` | A quota exists but no LimitRange defaults and the Pod has no `resources` | Create the LimitRange (step 1) or add `resources` |
| `exceeded quota` | The namespace total was reached | `kubectl describe resourcequota`; delete leftovers |
| PVC `Pending` forever | No Pod uses it yet (`WaitForFirstConsumer`), or no StorageClass | Create a Pod that mounts it; `kubectl get storageclass` |
| Ingress returns `404` | Host header does not match, or no controller | Check `host:`; `kubectl get pods -n kube-system` for Traefik |
| `OOMKilled` does not appear | Image cannot allocate memory as expected | Lower the limit or raise `--vm-bytes` |

## How to undo

```bash
kubectl delete namespace practice-mid
```

## Exercises to go further

1. Add a `startupProbe` to the `ready-demo` Deployment and explain what it delays.
2. Create a Deployment that mounts the Secret as **files** and read them inside the Pod.
3. Make the `ResourceQuota` allow only 2 Pods and scale a Deployment to 5. Read the events of the ReplicaSet.
4. Add a second host and path to the Ingress (`/api` to another Service).
5. Turn `ready-demo` into a StatefulSet with a headless Service and reach `ready-demo-0` by its stable name.

## References

- [Kubernetes: intermediate concepts](../concepts/kubernetes-intermediate-concepts.md)
- [Resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) and [limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
