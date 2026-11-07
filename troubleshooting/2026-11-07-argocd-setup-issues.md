# Issues found while installing ArgoCD and onboarding fintracker

- **Date:** 2026-11-07
- **Namespace:** `cicd-system`, `fintracker-dev`
- **Component:** ArgoCD v3.3.8 (Kustomize install), argocd CLI v3.3.8, fintracker CI/CD (GitHub Actions + python-semantic-release 10.5.3)
- **Status:** resolved

This record covers everything that went wrong *after* the first failed install, which is
documented separately in [`2026-11-07-argocd-resource-quota.md`](2026-11-07-argocd-resource-quota.md).
Each section is a separate issue, in the order they showed up.

---

## 1. Install instructions still described the old Helm install

### Symptom

The manifests in `manifests/ci-cd/` had already been rewritten for Kustomize, but the
ArgoCD section of `k3s-commands.md` still said:

```bash
helm install argocd argo/argo-cd --namespace cicd-system --values manifests/ci-cd/argocd-values.yaml --wait
kubectl apply -f manifests/ci-cd/argocd-ingress.yaml
```

Neither `argocd-values.yaml` nor `argocd-ingress.yaml` existed. Following the doc would
fail, or worse, reinstall ArgoCD with Helm next to the Kustomize resources.

### Cause

When the install moved from Helm to Kustomize, only the manifests were updated. The doc,
and the ingress that the Helm values used to create, were left behind.

### Solution

- Added `manifests/ci-cd/argocd-ingress.yaml` (Traefik, host `argocd.local`, backend
  `argocd-server` port `http`) and listed it in `kustomization.yaml`.
- Rewrote the ArgoCD section of `k3s-commands.md` for Kustomize: preview, watch commands,
  install, wait, verify, `/etc/hosts`, initial password, upgrade and uninstall.
- Checked the build before applying: `kubectl kustomize manifests/ci-cd/` builds 62
  resources, every container has CPU/memory set, `server.insecure: "true"` is present.

---

## 2. Server-side dry-run fails with `namespaces "cicd-system" not found`

### Symptom

```bash
kubectl apply -k manifests/ci-cd/ --server-side --dry-run=server
```

```
namespace/cicd-system serverside-applied (server dry run)
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io serverside-applied (server dry run)
...
Error from server (NotFound): namespaces "cicd-system" not found
Error from server (NotFound): namespaces "cicd-system" not found
...
```

### Understanding the problem

This is **not a real error**. A server dry-run sends each object to the API server, which
validates it and runs admission, but **does not save anything**. So the `Namespace` object is
"created" only in the dry-run, and when the next object (a ServiceAccount in
`cicd-system`) is validated, the namespace really does not exist yet.

- Cluster-scoped objects (Namespace, CRDs, ClusterRoles, ClusterRoleBindings) passed.
- Every namespaced object failed only with `NotFound` on the namespace.

### Solution

Nothing to fix. The real apply creates the namespace first, and everything else after it.
To get a clean dry-run on a fresh cluster, create the namespace first
(`kubectl apply -f manifests/ci-cd/namespace.yaml`), or read the dry-run output and accept
`NotFound` on the namespace as the only error.

---

## 3. `FailedMount` warning right after the install

### Symptom

```
Warning   FailedMount   pod/argocd-applicationset-controller-...   MountVolume.SetUp failed for volume
"argocd-cmd-params-cm" : failed to sync configmap cache: timed out waiting for the condition
```

### Understanding the problem

The kubelet (the agent on the node that starts containers) keeps a local cache of the
ConfigMaps and Secrets the pods mount. When a lot of objects are created at the same time,
like a full ArgoCD install, a pod can be scheduled before the kubelet's cache has the
ConfigMap. The mount fails once, the kubelet retries, and the pod starts normally.

### Solution

Nothing to fix. A few seconds later all 7 pods were `Running` with 0 restarts. Only worry
about `FailedMount` if it keeps repeating, or if the pod stays in `ContainerCreating`. In
that case check that the ConfigMap exists (`kubectl get cm -n cicd-system`).

---

## 4. The requests total in the docs was wrong (550m instead of 650m)

### Symptom

After the install, the quota showed:

```
Resource         Used    Hard
limits.cpu       2200m   3
requests.cpu     650m    1
```

But the previous record and the header of `argocd-resources.yaml` said the requests added
up to 550m.

### Cause

A mistake in the sum. The real requests of the main containers are:

| Component | requests.cpu |
|---|---|
| argocd-application-controller | 250m |
| argocd-server | 100m |
| argocd-repo-server | 100m |
| argocd-applicationset-controller | 50m |
| argocd-notifications-controller | 50m |
| argocd-redis | 50m |
| argocd-dex-server | 50m |
| **Total** | **650m** |

Init containers do not add to the total. They run *before* the main containers and stop, so
the quota counts, per pod, the larger of "the biggest init container" and "the sum of the
main containers".

### Solution

Fixed the number in `argocd-resources.yaml` and in the quota record. 650m still fits in
`requests.cpu: "1"`.

---

## 5. argocd CLI: `Argo CD server address unspecified`

### Symptom

```bash
argocd account update-password
```

```
{"level":"fatal","msg":"Argo CD server address unspecified","time":"2026-11-07T12:35:46-03:00"}
```

### Diagnosis

```bash
ls ~/.config/argocd/      # No such file or directory
argocd context            # No local configuration found
grep argocd /etc/hosts    # (nothing)
```

### Cause

The CLI had never logged in successfully, so it had no saved server address. Most CLI
commands read the server from `~/.config/argocd/config`, which `argocd login` writes. The
login through the ingress could not work because `argocd.local` did not resolve to anything.

### Understanding the problem

- **The CLI keeps a session.** `argocd login <server>` saves the server, the token and the
  connection options in a "context". Later commands use the current context. Without it they
  fail with "server address unspecified".
- **Why `--plaintext --grpc-web`.** The server runs with `server.insecure: "true"`, so it
  speaks plain HTTP (Traefik would handle TLS). `--plaintext` tells the CLI not to use TLS.
  The CLI talks gRPC, and Traefik forwards it more reliably as gRPC-Web (gRPC carried over
  normal HTTP requests), which is what `--grpc-web` does.
- **The port-forward mode does not save the server.** With
  `argocd login --port-forward --port-forward-namespace cicd-system`, every later command
  needs the same flags (or `ARGOCD_OPTS`), or it fails with the same error.

### Solution

```bash
echo "192.168.18.2  argocd.local" | sudo tee -a /etc/hosts     # Traefik LoadBalancer IP
argocd login argocd.local --plaintext --grpc-web --username admin
argocd account update-password
```

---

## 6. CI and ArgoCD would fight over the fintracker image

### Symptom

When the `fintracker` Application was created (path `k8s/` of the fintracker repo), the
manifests in git and the cluster did not agree on the image:

| Where | Image |
|---|---|
| `k8s/app-deployment.yaml` in git | `jvrochadev/fintracker:latest` |
| Deployment in the cluster | `jvrochadev/fintracker:1.2.0` |

### Cause

The GitHub Actions `deploy` job changed the cluster directly, after each release:

```bash
kubectl set image deployment/fintracker-deployment fintracker-web=jvrochadev/fintracker:<version> -n fintracker-dev
```

That change never went back to git. With two sources of truth:

- a manual **Sync** in ArgoCD would set the image back to `:latest`;
- with **Auto Sync + Self Heal**, ArgoCD would undo every CI deploy within minutes, because
  it puts the cluster back to what git says.

### Understanding the problem

GitOps means the git repository is the only description of what should run. ArgoCD keeps
comparing git with the cluster. Any change made only in the cluster (`kubectl set image`,
`kubectl edit`, `kubectl scale`) shows up as drift, and with Self Heal it is reverted. So
the CI should not deploy to the cluster any more: it should **write the new version into
git** and let ArgoCD deploy it.

### Solution

In the fintracker repo (commit `ae471b8`):

1. New `k8s/kustomization.yaml` that lists the manifests (not the git-ignored Secret) and
   pins the image:
   ```yaml
   images:
     - name: jvrochadev/fintracker
       newTag: "1.2.0"
   ```
2. `pyproject.toml`: `version_variables = ["k8s/kustomization.yaml:newTag"]`. On each
   release, python-semantic-release writes the new version in both `pyproject.toml` and
   `newTag`, in the same release commit and tag (`v<version>`).
3. `.github/workflows/deploy.yaml`: removed the `deploy` job (install kubectl,
   `set image`, `rollout status`). The pipeline is now `test` → `versioning` →
   `build-and-push`.

Verified:

- Locally, with python-semantic-release 10.5.3, the `newTag` pattern replaces
  `newTag: "1.2.0"` with `newTag: "1.3.0"`, and `semantic-release --noop version` reads the
  config without errors.
- `kubectl kustomize k8s/` renders `image: jvrochadev/fintracker:1.2.0`.
- ArgoCD detected the new revision as **Kustomize** (it was Directory before). The
  Application spec has no `directory:` field, so the source type is detected automatically.
- `argocd app diff` shows no image difference any more (see issue 7 for what it does show).

### Known trade-off

The release commit (with the new `newTag`) reaches git *before* `build-and-push` publishes
the image. If ArgoCD syncs in between, the new pod is in `ImagePullBackOff` for a few
minutes. With 2 replicas, the rolling update keeps the old pods running until the new one
is ready (`maxUnavailable` 25% of 2 rounds down to 0), and the kubelet keeps retrying the
pull, so it recovers without anyone doing anything once the image exists.

---

## 7. Every fintracker resource shows OutOfSync, with no real difference

### Symptom

Right after the Application was created, all 4 resources were `OutOfSync`, even after the
image fix:

```
ConfigMap/fintracker-configmap OutOfSync
Namespace/fintracker-dev OutOfSync
Service/fintracker-svc OutOfSync
Deployment/fintracker-deployment OutOfSync
```

### Diagnosis

```bash
argocd app diff cicd-system/fintracker
```

```
===== apps/Deployment fintracker-dev/fintracker-deployment ======
>     argocd.argoproj.io/tracking-id: fintracker:apps/Deployment:fintracker-dev/fintracker-deployment
```

Every resource showed the same thing: only the tracking annotation was missing.

### Understanding the problem

ArgoCD v3 marks each resource it manages with the annotation
`argocd.argoproj.io/tracking-id`. That is how it knows which Application owns which object.
These resources were created months ago with `kubectl apply`, so they do not have it, and
ArgoCD reports them as different from git even though their content is the same.

### Solution

The first **Sync** adds the annotation. On the Deployment it goes into `metadata`, not into
the pod template, so the sync does **not** restart the pods. After that the resources show
`Synced`.

---

## Lessons

- **Update the docs in the same change as the manifests.** A doc that points to files that
  no longer exist is worse than no doc.
- **Read a server dry-run knowing what it cannot do.** It does not save anything, so objects
  that depend on a new namespace fail with `NotFound`.
- **One-off warning events at startup are normal.** What matters is whether they repeat and
  whether the pods become `Ready`.
- **Check sums in docs against the cluster.** `kubectl describe resourcequota` shows the real
  totals.
- **The argocd CLI needs a successful `argocd login`** (and a resolvable server name) before
  any other command.
- **With ArgoCD, nothing else should change the cluster.** CI writes versions into git;
  ArgoCD applies them. A `kubectl set image` in a pipeline is drift.
- **Adopting existing resources makes them OutOfSync at first.** Check `argocd app diff`:
  if it only shows `tracking-id`, the first Sync is safe.
