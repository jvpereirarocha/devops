# ArgoCD without application-controller and without services (ResourceQuota)

- **Date:** 2026-11-07
- **Namespace:** `cicd-system`
- **Component:** ArgoCD v3.3.8
- **Status:** resolved (ArgoCD removed and manifests rewritten with Kustomize)

## Symptom

- `kubectl get deploy -A` showed the ArgoCD deployments in the `cicd-system` namespace.
- `kubectl get svc -A` did not show the `cicd-system` namespace: there was no Service in it. `get svc -A` only lists namespaces that have at least one service.
- The `argocd-application-controller` StatefulSet was at `0/1`, and no `Application` existed in the cluster.

## Diagnosis

```bash
kubectl get all -n cicd-system
kubectl get events -n cicd-system --sort-by=.lastTimestamp
kubectl describe resourcequota -n cicd-system
helm history argocd -n cicd-system
```

Two problems were found.

**1. The Helm installation had failed halfway through:**

```
1  Sat Apr 25 15:47:32 2026  failed  argo-cd-9.5.4  Release "argocd" failed: context canceled
```

`context canceled` means the command was interrupted (Ctrl+C or a `--wait` timeout). The 4 services were in the release manifest, but they did not exist in the cluster.

**2. The application-controller was blocked by the ResourceQuota:**

```
create Pod argocd-application-controller-0 in StatefulSet argocd-application-controller failed error:
pods "argocd-application-controller-0" is forbidden: exceeded quota: cicd-system-quota,
requested: limits.cpu=500m, used: limits.cpu=1600m, limited: limits.cpu=2
```

## Understanding the quota problem

### Requests and limits

Each container can declare two CPU values:

- **request:** how much CPU the container is guaranteed. Kubernetes uses this value to decide which node the pod fits on.
- **limit:** the most the container can use. If it goes above that, it is throttled.

For example, `argocd-server` has `requests.cpu: 100m` and `limits.cpu: 500m`. It usually uses little, but it can reach half a core. The `m` unit means millicore: `1000m` is 1 core, and `500m` is half a core.

### ResourceQuota: a ceiling for the whole namespace

The `cicd-system-quota` limits the **sum** of all pods in the namespace:

```yaml
hard:
  requests.cpu: "1"     # sum of the requests of all pods: at most 1 core
  limits.cpu: "2"       # sum of the limits of all pods: at most 2 cores
```

Before creating a pod, Kubernetes checks: "adding this pod to the ones that already exist, does it go over the ceiling?". If it does, it **refuses to create the pod**. That is what the error above says: "this pod asks for 500m, the others already use 1600m, and 1600 + 500 = 2100m goes over the 2000m ceiling".

### LimitRange: the default value for containers that declare nothing

The `cicd-system-limits` (LimitRange) sets default values for containers that do not declare `resources`. The default is `limits.cpu: 500m`. The Helm values did not set resources for the application-controller, so it got this default.

### Why the sum went over 2

| Component | limits.cpu |
|---|---|
| argocd-server | 500m |
| argocd-repo-server | 500m |
| argocd-applicationset-controller | 200m |
| argocd-notifications-controller | 200m |
| argocd-redis | 200m |
| **Subtotal (the ones that started)** | **1600m** |
| argocd-application-controller | 500m (LimitRange default) |
| **Total needed** | **2100m**, does not fit in 2000m |

The application-controller, which is ArgoCD's main component, was left out. It was last in line and there was no room left in the quota.

## Solution

1. ArgoCD was removed completely: Helm release, CRDs, ClusterRoles and namespace.
2. The installation was redone without Helm, with Kustomize, in `manifests/ci-cd/`:
   - `argocd-resources.yaml` sets CPU and memory for **every** container, including the application-controller and the init containers, so nothing depends on the LimitRange default.
   - The quota's `limits.cpu` went from `"2"` to `"3"`.

### Why 3

In the new installation, the sum of the limits is **2200m**. The application-controller and Dex, which comes in the official manifest, are now included. With the ceiling at `3`, there are 800m of headroom.

That headroom matters during a rolling update. The new pod starts before the old one is removed, so for a few seconds both count against the quota at the same time. Without headroom, the update could get stuck for the same reason.

### Does this load the server?

No. Raising `limits.cpu` does not reserve CPU on the node; it only allows the pods together to *be able to* reach 3 cores. What Kubernetes actually reserves are the requests, and those stay low: 650m in total, within `requests.cpu: "1"`, which did not change. At the time, the node was using about 5% CPU.

## Lessons

- **When defining a ResourceQuota, add up the limits of every component**, including the ones that do not declare resources and get the LimitRange default, and leave headroom for rolling updates.
- **If a Deployment or StatefulSet pod does not show up, look at the namespace events** (`kubectl get events -n <ns>`). The quota error only shows up there, not in `get pods`.
- **Do not interrupt an installation halfway.** An interrupted release is left with missing resources. Check the result with `helm history` or `kubectl get all -n <ns>`.
- **`kubectl get <resource> -A` only shows namespaces that have that resource.** A namespace missing from the list does not mean it does not exist.
