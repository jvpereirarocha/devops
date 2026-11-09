# How to limit the old ReplicaSets kept by a Deployment

> **Summary:** understand why a Deployment keeps old, empty ReplicaSets after every rollout
> (even when ArgoCD does the deploy), and cap them with `revisionHistoryLimit` through git so
> only the running version and the last two versions stay in the cluster.

- **Last updated:** 2026-11-09
- **Components:** k3s v1.34, ArgoCD v3.3.8 (automated sync with self-heal), Kustomize (built into `kubectl`)
- **Level:** beginner

## Context

After a few deploys of an API through ArgoCD, `kubectl get rs` showed six ReplicaSets for the
same Deployment. Only one had pods; the other five had `DESIRED 0`:

```
NAME                               DESIRED   CURRENT   READY   AGE    IMAGES
fintracker-deployment-7ff4d7b585   0         0         0       222d   jvrochadev/fintracker:latest
fintracker-deployment-6756d7b7b5   0         0         0       221d   jvrochadev/fintracker:latest
fintracker-deployment-8956c898d    0         0         0       199d   jvrochadev/fintracker:latest
fintracker-deployment-7dcfb5f4cb   0         0         0       4d2h   jvrochadev/fintracker:latest
fintracker-deployment-7758ff6fc8   0         0         0       3d23h  jvrochadev/fintracker:1.2.0
fintracker-deployment-c98b6c9c9    2         2         2       10m    jvrochadev/fintracker:1.3.0
```

It looks like ArgoCD is leaving garbage behind, but it is not. This is the normal behavior of
the Deployment controller, and it is there on purpose.

### Why the old ReplicaSets stay

A **ReplicaSet** is the object that keeps N identical pods running. A **Deployment** does not
manage pods directly: it manages ReplicaSets, one per version of the pod template.

1. **Every change to the pod template creates a new ReplicaSet.** When the image tag (or an
   env var, a probe, a label in `spec.template`) changes, the Deployment computes a new
   `pod-template-hash` and creates a ReplicaSet with that hash in its name.
2. **A rolling update scales the old ReplicaSet to zero; it does not delete it.** The new
   ReplicaSet grows and the old one shrinks step by step (`maxSurge` / `maxUnavailable`).
   When the old one reaches `replicas: 0`, the rollout is done, and the object is still there.
3. **The old ReplicaSet is the rollback.** It still holds the full pod template of that
   version (image, env, probes). `kubectl rollout undo` just scales it back up: no rebuild,
   no manifest needed. Each ReplicaSet is one line of `kubectl rollout history`.
4. **`spec.revisionHistoryLimit` decides how many old ones to keep.** The default is **10**.
   When there are more old ReplicaSets than the limit, the controller deletes the oldest ones.
   With only five old ones, nothing had been cleaned up yet.

### Why ArgoCD does not delete them

ArgoCD only manages what is in git: the Deployment, the Service, the ConfigMap. The
ReplicaSets are **children** of the Deployment (their `metadata.ownerReferences` point to it)
and never appear in git. ArgoCD shows them in its UI tree, but they are never candidates for
`prune`. Only the Deployment controller handles them.

### What they cost

Almost nothing. A ReplicaSet with `replicas: 0` is just an object in the cluster datastore: no
pods, no CPU, no memory, and it does not count against a ResourceQuota for pods. The real cost
is noise in `kubectl get rs` and in the ArgoCD UI, plus a rollback history that is longer than
anyone will use.

## Prerequisites

- A Deployment whose manifest lives in git and is synced by an ArgoCD Application
  (see [How to deploy ArgoCD on the Kubernetes cluster](how-to-deploy-argocd-on-the-kubernetes-cluster.md)).
- `kubectl` pointing at the cluster.

## Overview

1. Look at the current ReplicaSets and the rollout history.
2. Add `revisionHistoryLimit` to the Deployment manifest in git.
3. Check the change against the live cluster with `kubectl diff`.
4. Commit and push; ArgoCD applies it.
5. Check that the old ReplicaSets were removed and no pod restarted.

## Step by step

### 1. Look at the current state

```bash
kubectl get deploy <deployment> -n <namespace> \
  -o jsonpath='{.spec.revisionHistoryLimit}{"\n"}'
kubectl rollout history deploy/<deployment> -n <namespace>
kubectl get rs -n <namespace> -l app=<app-label> \
  -o custom-columns=NAME:.metadata.name,REV:.metadata.annotations.deployment\\.kubernetes\\.io/revision,IMAGE:.spec.template.spec.containers[0].image,DESIRED:.spec.replicas \
  --sort-by=.metadata.annotations.deployment\\.kubernetes\\.io/revision
```

**Why:** the annotation `deployment.kubernetes.io/revision` on each ReplicaSet is its revision
number. It tells you which ones the controller will delete first (the lowest numbers).

Expected output: the limit is `10` if the manifest never set it, and one ReplicaSet per
revision, only the highest one with `DESIRED` greater than zero.

### 2. Set `revisionHistoryLimit` in the manifest

Edit the Deployment in the git repository that ArgoCD watches (in this homelab:
`k8s/app-deployment.yaml` in the application repository):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <deployment>
  namespace: <namespace>
spec:
  replicas: 2
  # Keep only the last 2 old ReplicaSets (3 in total with the current one) for
  # `kubectl rollout undo`; the default (10) piles up idle ReplicaSets.
  # The normal rollback path is reverting the commit in git.
  revisionHistoryLimit: 2
  selector:
    ...
```

**Why in git and not with `kubectl patch`:** with ArgoCD's self-heal on, a change made only in
the cluster is reverted on the next sync. Git is the source of truth.

**Important:** the limit counts only the **old** ReplicaSets. `revisionHistoryLimit: 2` means
three ReplicaSets in total: the running one plus two previous versions.

### 3. Check the change against the cluster

```bash
kubectl diff -k k8s/ | grep -E '^[+-] ' | grep -v tracking-id
```

**Why:** `kubectl diff` asks the API server what would change, without changing anything.
Kustomize-based repos use `-k`; plain YAML uses `-f`.

Expected output:

```
-  generation: 10
+  generation: 11
-  revisionHistoryLimit: 10
+  revisionHistoryLimit: 2
```

The `argocd.argoproj.io/tracking-id` annotation always shows up as removed in this diff
because ArgoCD adds it when it syncs and the plain manifest does not have it. It is safe to
ignore. If you see anything under `spec.template`, stop: that would trigger a new rollout.

### 4. Commit and push

```bash
git add k8s/app-deployment.yaml
git commit -m "chore(k8s): keep only the last 2 old ReplicaSets of the API deployment"
git push origin main
```

**Why `chore`:** the repository uses semantic-release with conventional commits. A `chore`
commit does not cut a release, so the pipeline runs only the tests and no new image is built.
ArgoCD (automated sync) picks up the new commit on its next poll, about 3 minutes by default.

To follow along in another terminal:

```bash
kubectl get rs -n <namespace> -l app=<app-label> -w
```

### 5. Verify

```bash
kubectl get deploy <deployment> -n <namespace> \
  -o jsonpath='limit={.spec.revisionHistoryLimit} revision={.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}'
kubectl get rs -n <namespace> -l app=<app-label>
kubectl get pods -n <namespace> -l app=<app-label>
```

Expected output:

- `limit=2` and the **same** revision number as before. `revisionHistoryLimit` is outside
  `spec.template`, so changing it does not create a new revision.
- Three ReplicaSets: the current one with pods and the two most recent old ones with `0`.
- The pods keep their old `AGE` and restart count: nothing was restarted.

## Technical decisions

### Keep two old ReplicaSets

- **Decision:** `revisionHistoryLimit: 2`.
- **Why:** with GitOps, the normal rollback is `git revert` of the release commit, and ArgoCD
  deploys the previous version. `kubectl rollout undo` is only for emergencies (for example,
  when git or CI is down), and the last one or two versions cover that.
- **Alternatives considered:**
  - Keep the default (10): works, but keeps months of empty ReplicaSets nobody will use.
  - `revisionHistoryLimit: 0`: deletes every old ReplicaSet and makes `kubectl rollout undo`
    impossible.
  - Delete the old ReplicaSets by hand with `kubectl delete rs`: they come back after the next
    deploys, and you may delete the one you need for a rollback.
- **Trade-offs / risks:**
  - You cannot `rollout undo` to versions older than two releases. A `git revert` still can.
  - A `kubectl rollout undo` in a cluster managed by ArgoCD with self-heal is reverted to what
    git says on the next sync. After an emergency undo, fix git right away (revert the commit),
    or turn off auto-sync for that Application while you work.

### A rollback is only as good as the image tag

- **Decision:** rely on versioned image tags (`1.2.0`, `1.3.0`) for rollback, not on `latest`.
- **Why:** an old ReplicaSet that points to `image:latest` with `imagePullPolicy: Always`
  pulls whatever `latest` is **today**. If CI pushes `latest` together with every release,
  "rolling back" to that ReplicaSet runs the current code. Only a ReplicaSet with a fixed tag
  really brings back the old version.
- **Trade-offs / risks:** right after this change, one of the two kept ReplicaSets still pointed
  to `latest`. It fixes itself after two more releases, when the history holds only tagged
  versions.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| The limit goes back to 10 a few minutes after a `kubectl patch` | ArgoCD self-heal reverts changes not in git | Change the manifest in git |
| Still more ReplicaSets than expected | The limit counts only old ReplicaSets; the current one is extra | Expect `revisionHistoryLimit + 1` |
| `rollout undo` to an old revision runs the current code | That ReplicaSet uses a mutable tag such as `latest` | Use versioned image tags |
| `rollout undo` worked, then the old version came back | ArgoCD synced git again | Revert the commit in git instead |
| A new rollout happened after the change | Something under `spec.template` also changed | Check `kubectl diff` before pushing |

## How to undo

Revert the commit and push. ArgoCD sets the limit back to the default:

```bash
git revert <commit-sha>
git push origin main
```

The ReplicaSets that were deleted do not come back; only new rollouts will build the history
again.

## References

- Kubernetes docs, Deployments, "Revision History Limit" and "Rolling Back a Deployment":
  https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes docs, Garbage collection and owner references:
  https://kubernetes.io/docs/concepts/architecture/garbage-collection/
- ArgoCD docs, Automated sync and self-heal:
  https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/
