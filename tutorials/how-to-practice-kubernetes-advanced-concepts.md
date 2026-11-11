# How to practice Kubernetes advanced concepts

> **Summary:** a hands-on lab covering RBAC with a ServiceAccount, a NetworkPolicy that isolates a database, autoscaling with an HPA, a PodDisruptionBudget with `cordon`, Pod Security enforcement, a CustomResourceDefinition and a Kustomize overlay. Each step proves the concept with a command that fails before the change and works after it (or the other way round).

- **Last updated:** 2026-11-11
- **Components:** any Kubernetes cluster (written for k3s, which includes metrics-server and a NetworkPolicy controller), `kubectl` with built-in Kustomize, public images `nginx`, `nginxinc/nginx-unprivileged`, `busybox` and `registry.k8s.io/hpa-example`
- **Level:** advanced
- **Theory:** [Kubernetes: advanced concepts](../concepts/kubernetes-advanced-concepts.md)
- **Do first:** [basic](how-to-practice-kubernetes-basic-concepts.md) and [intermediate](how-to-practice-kubernetes-intermediate-concepts.md) labs

## Context

These are the features that separate "it runs" from "it runs safely, scales and can be operated by a team". Run this lab on a **test** cluster. It is safe on a single node as long as you follow the warning in step 4.

## Prerequisites

- Cluster-admin access (steps 6 and 7 create cluster-wide objects).
- metrics-server working: `kubectl top nodes` prints numbers (step 3).
- A CNI that enforces NetworkPolicies (k3s does by default; some CNIs do not, which makes step 2 appear to "not work").
- Two terminals.

## Overview

1. RBAC: a ServiceAccount that can only read Pods.
2. NetworkPolicy: only the API may reach the database.
3. HPA: scale on CPU load.
4. PodDisruptionBudget and `cordon`.
5. Pod Security: reject an unsafe Pod.
6. CRD: teach the API a new kind.
7. Kustomize: one base, two environments.
8. Clean up.

## Step by step

### 1. RBAC: least privilege

```bash
kubectl create namespace practice-adv
kubectl config set-context --current --namespace=practice-adv

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-viewer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
EOF
```

**Before the binding**, the identity can do nothing:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:practice-adv:pod-viewer
```

Expected: `no`. Now bind the Role:

```bash
kubectl create rolebinding view-pods --role=pod-reader --serviceaccount=practice-adv:pod-viewer

kubectl auth can-i list pods   --as=system:serviceaccount:practice-adv:pod-viewer   # yes
kubectl auth can-i delete pods --as=system:serviceaccount:practice-adv:pod-viewer   # no
kubectl auth can-i list pods -n kube-system --as=system:serviceaccount:practice-adv:pod-viewer   # no
```

**What you learned:** permissions are explicit (`yes` only for what the Role lists), they are limited by verb, and a Role only works inside its own namespace (the `kube-system` check is `no`).

**See it from inside a Pod:**

```bash
kubectl run rbac-test --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"pod-viewer","containers":[{"name":"rbac-test","image":"busybox:1.36","command":["sleep","600"]}]}}'
kubectl wait --for=condition=Ready pod/rbac-test --timeout=60s
kubectl exec rbac-test -- sh -c '
  CA=/var/run/secrets/kubernetes.io/serviceaccount
  TK=$(cat $CA/token)
  wget -qO- --header "Authorization: Bearer $TK" --ca-certificate=$CA/ca.crt \
    https://kubernetes.default.svc/api/v1/namespaces/practice-adv/pods | head -c 200; echo
  wget -qO- --header "Authorization: Bearer $TK" --ca-certificate=$CA/ca.crt \
    https://kubernetes.default.svc/api/v1/namespaces/kube-system/pods 2>&1 | tail -1'
kubectl delete pod rbac-test --now
```

**Expected:** the first call prints the start of a `PodList` JSON; the second ends with `403 Forbidden`.

### 2. NetworkPolicy: isolate the database

Create a "database" and two clients:

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: db }
spec:
  replicas: 1
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
        - { name: nginx, image: nginx:1.27, ports: [{ containerPort: 80 }] }
---
apiVersion: v1
kind: Service
metadata: { name: db }
spec:
  selector: { app: db }
  ports: [{ port: 80 }]
---
apiVersion: v1
kind: Pod
metadata: { name: api, labels: { app: api } }
spec:
  containers: [{ name: c, image: busybox:1.36, command: ["sleep", "3600"] }]
---
apiVersion: v1
kind: Pod
metadata: { name: other, labels: { app: other } }
spec:
  containers: [{ name: c, image: busybox:1.36, command: ["sleep", "3600"] }]
EOF
kubectl wait --for=condition=Ready pod/api pod/other --timeout=90s
```

**Before the policy**, both can connect:

```bash
kubectl exec api   -- wget -qO- -T 3 http://db | head -2
kubectl exec other -- wget -qO- -T 3 http://db | head -2
```

Apply the policy:

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-only-from-api }
spec:
  podSelector:
    matchLabels: { app: db }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: api }
      ports:
        - port: 80
EOF
```

**After the policy:**

```bash
kubectl exec api   -- wget -qO- -T 3 http://db | head -2     # still works
kubectl exec other -- wget -qO- -T 3 http://db | head -2     # times out
```

**Expected:** `api` still gets the nginx page; `other` fails with `download timed out`. Once a Pod is selected by a policy, everything not allowed is dropped.

> If `other` still connects, your CNI does not enforce NetworkPolicies. The object is accepted but ignored.

### 3. HPA: autoscaling on CPU

```bash
kubectl top nodes        # must print numbers; if not, metrics-server is missing

kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: php-apache }
spec:
  replicas: 1
  selector: { matchLabels: { app: php-apache } }
  template:
    metadata: { labels: { app: php-apache } }
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: 100m, memory: 64Mi }
            limits:   { cpu: 500m, memory: 128Mi }
---
apiVersion: v1
kind: Service
metadata: { name: php-apache }
spec:
  selector: { app: php-apache }
  ports: [{ port: 80 }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: php-apache }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: php-apache }
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 50 }
EOF
```

**Why the CPU request matters:** "50 % utilization" means 50 % *of the request* (50 m here). Without a request the HPA shows `<unknown>`.

In the **second terminal**:

```bash
kubectl get hpa php-apache -w
```

In the first, generate load:

```bash
kubectl run load --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://php-apache >/dev/null; done'
```

**Expected:** within 1-2 minutes `TARGETS` rises above 50 % and `REPLICAS` grows (up to 5). Stop the load and wait; scale-down is deliberately slow (about 5 minutes) to avoid flapping.

```bash
kubectl delete pod load --now
```

### 4. PodDisruptionBudget and cordon

```bash
kubectl create deployment pdb-demo --image=nginx:1.27 --replicas=3
kubectl apply -f - <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: pdb-demo }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: pdb-demo }
EOF
kubectl get pdb pdb-demo
```

**Expected:** `ALLOWED DISRUPTIONS 1` (3 Pods, at least 2 must stay). A `drain` would evict at most one Pod at a time and wait for its replacement.

> **WARNING: on a single-node cluster never run `kubectl drain` on your only node.** It would evict every Pod, including the ones the cluster needs. Practice with `cordon`, which is harmless:

```bash
kubectl cordon <NODE_NAME>
kubectl get nodes                       # STATUS: Ready,SchedulingDisabled
kubectl scale deployment pdb-demo --replicas=5
kubectl get pods -l app=pdb-demo        # the new Pods stay Pending
kubectl uncordon <NODE_NAME>
kubectl get pods -l app=pdb-demo        # they start
```

**What you learned:** `cordon` stops *new* Pods from arriving without touching running ones; `drain` is `cordon` plus eviction, and PDBs limit that eviction.

### 5. Pod Security: reject an unsafe Pod

Make the namespace enforce the `restricted` profile:

```bash
kubectl create namespace practice-adv-secure
kubectl label namespace practice-adv-secure pod-security.kubernetes.io/enforce=restricted

kubectl run unsafe --image=nginx:1.27 -n practice-adv-secure
```

**Expected:** `Error from server (Forbidden): pods "unsafe" is forbidden: violates PodSecurity "restricted:latest"` followed by the list of problems (privilege escalation, capabilities, non-root, seccomp). The default `nginx` image runs as root, so it is refused.

Now a compliant Pod:

```bash
kubectl apply -n practice-adv-secure -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: safe }
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: web
      image: nginxinc/nginx-unprivileged:1.27
      ports: [{ containerPort: 8080 }]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
EOF
kubectl get pod safe -n practice-adv-secure
```

**Expected:** `Running`. The image listens on 8080 and runs as a non-root user, and the manifest drops privileges.

### 6. CRD: teach the API a new kind

```bash
kubectl apply -f - <<'EOF'
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.practice.example.com
spec:
  group: practice.example.com
  scope: Namespaced
  names: { plural: backups, singular: backup, kind: Backup }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: [schedule]
              properties:
                schedule: { type: string }
                keep: { type: integer, minimum: 1 }
EOF
kubectl get crd backups.practice.example.com
```

Create an object of the new kind, and try an invalid one:

```bash
kubectl apply -f - <<'EOF'
apiVersion: practice.example.com/v1
kind: Backup
metadata: { name: nightly }
spec: { schedule: "0 3 * * *", keep: 7 }
EOF
kubectl get backups
kubectl explain backup.spec

kubectl apply -f - <<'EOF'
apiVersion: practice.example.com/v1
kind: Backup
metadata: { name: broken }
spec: { keep: 0 }
EOF
```

**Expected:** `nightly` is created and listed like any built-in object; `broken` is rejected (`schedule: Required value` and `keep ... should be greater than or equal to 1`). The API server validates your kind from the schema.

**Important:** nothing *acts* on a `Backup` yet. A CRD only stores data. An **operator** is a controller that watches these objects and does the work. That is how ArgoCD's `Application` works.

### 7. Kustomize: one base, two environments

```bash
mkdir -p /tmp/practice-kustomize/base /tmp/practice-kustomize/overlays/{dev,prod}
cd /tmp/practice-kustomize

cat > base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: { name: site }
spec:
  replicas: 1
  selector: { matchLabels: { app: site } }
  template:
    metadata: { labels: { app: site } }
    spec:
      containers:
        - { name: web, image: nginx:1.26 }
EOF
cat > base/kustomization.yaml <<'EOF'
resources: [deployment.yaml]
EOF
cat > overlays/dev/kustomization.yaml <<'EOF'
resources: [../../base]
namePrefix: dev-
EOF
cat > overlays/prod/kustomization.yaml <<'EOF'
resources: [../../base]
namePrefix: prod-
replicas:
  - { name: site, count: 3 }
images:
  - { name: nginx, newTag: "1.27" }
EOF

kubectl kustomize overlays/dev  | grep -E 'name:|replicas:|image:'
kubectl kustomize overlays/prod | grep -E 'name:|replicas:|image:'
```

**Expected:** `dev-site` with 1 replica and `nginx:1.26`; `prod-site` with 3 replicas and `nginx:1.27`. Same base, different results, no copied YAML.

Preview the cluster change before applying, then apply the dev one:

```bash
kubectl diff -k overlays/dev | head -20
kubectl apply -k overlays/dev
kubectl get deployment
```

`kubectl diff` and `kubectl apply --dry-run=server` are the habit to build before touching a real cluster.

### 8. Clean up

```bash
cd ~ && rm -rf /tmp/practice-kustomize
kubectl config set-context --current --namespace=default
kubectl delete namespace practice-adv practice-adv-secure
kubectl delete crd backups.practice.example.com
```

Deleting the CRD also deletes every `Backup` object. Cluster-wide objects (CRDs) are not removed with a namespace, so remember this step.

## Technical decisions

### Lab on a test cluster, `cordon` instead of `drain`

- **Decision:** practice disruption with `cordon` only.
- **Why:** on one node, `drain` evicts everything and the PDB cannot protect you because there is nowhere to reschedule.
- **Alternatives considered:** a second node or a throwaway cluster (k3d/kind) to run a real drain.
- **Trade-offs / risks:** `cordon` shows the scheduling effect but not eviction; try `drain` on a multi-node test cluster.

### `restricted` Pod Security on a dedicated namespace

- **Decision:** enforce `restricted` only on `practice-adv-secure`.
- **Why:** it shows the rejection without breaking other workloads.
- **Alternatives considered:** `warn`/`audit` modes first (report violations without blocking), the safer way to roll it out on a live namespace.
- **Trade-offs / risks:** many public images run as root and will be refused; plan image changes before enforcing.

### CRD without an operator

- **Decision:** create the CRD and objects only.
- **Why:** it isolates the idea: the API learns a kind and validates it.
- **Alternatives considered:** installing a real operator (more moving parts than a lab needs).
- **Trade-offs / risks:** it can look like "nothing happens"; that is the point, a controller is what acts.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| HPA `TARGETS <unknown>/50%` | No CPU `requests`, or metrics-server missing/slow | Add requests; `kubectl top pods`; wait a minute |
| `other` Pod can still reach `db` | CNI does not enforce NetworkPolicy | Use a CNI that does (k3s' default does) |
| `auth can-i` says `no` after binding | Wrong namespace or ServiceAccount name in the binding | `kubectl describe rolebinding view-pods` |
| New Pods `Pending` after cordon | Expected: the only node is unschedulable | `kubectl uncordon <NODE_NAME>` |
| CRD object rejected with `no matches for kind` | CRD not yet established | Wait a few seconds and retry |
| Pod rejected by PodSecurity | Image runs as root or lacks `securityContext` | Add the fields from step 5 or use an unprivileged image |

## How to undo

```bash
kubectl delete namespace practice-adv practice-adv-secure
kubectl delete crd backups.practice.example.com
kubectl uncordon <NODE_NAME>
kubectl delete -k /tmp/practice-kustomize/overlays/dev   # if the folder still exists
```

## Exercises to go further

1. Turn the `Role` into a `ClusterRole` + `ClusterRoleBinding` and check `kube-system` again.
2. Write a NetworkPolicy that denies all ingress in the namespace, then allow only what the `api` needs. Add an *egress* rule too.
3. Add `behavior.scaleDown.stabilizationWindowSeconds: 60` to the HPA and compare the scale-down time.
4. Put the `restricted` label as `warn` instead of `enforce` and see what `kubectl run` says.
5. Add a `topologySpreadConstraints` block to `pdb-demo`. On one node, what happens with `whenUnsatisfiable: DoNotSchedule`?
6. Read [How to deploy ArgoCD](how-to-deploy-argocd-on-the-kubernetes-cluster.md), then change a field of a managed Deployment with `kubectl edit` and watch self-heal revert it.

## References

- [Kubernetes: advanced concepts](../concepts/kubernetes-advanced-concepts.md)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [HPA walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Disruptions and PDBs](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Kustomize](https://kubectl.docs.kubernetes.io/references/kustomize/)
