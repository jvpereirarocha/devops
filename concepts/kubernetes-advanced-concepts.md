# Kubernetes: advanced concepts

> **In one sentence:** the advanced part of Kubernetes is about who is allowed to do what, how the cluster scales and heals itself, how Pods are placed and isolated, and how you extend Kubernetes with your own objects and automate delivery with GitOps.

- **Last updated:** 2026-11-11
- **Related:** [hands-on tutorial](../tutorials/how-to-practice-kubernetes-advanced-concepts.md), [basic concepts](kubernetes-basic-concepts.md), [intermediate concepts](kubernetes-intermediate-concepts.md), [tutorial: ArgoCD](../tutorials/how-to-deploy-argocd-on-the-kubernetes-cluster.md), [tutorial: frontend with ArgoCD and GitHub Actions](../tutorials/how-to-deploy-a-frontend-with-argocd-and-a-github-actions-release-pipeline.md)

## The idea in plain words

By now your apps run, store data and are reachable. In a bigger or shared cluster new questions appear:

- *Who* may delete a Deployment? (permissions)
- What happens when traffic triples? (autoscaling)
- Can the database Pod run on the node with the fast disk? (placement)
- Can the frontend talk to the database directly? (network isolation)
- How do I keep every change in git and have the cluster follow it? (GitOps)

Analogy: the restaurant now has several chefs, many customers and health inspectors. You need **badges** (RBAC), **hiring more cooks when it is busy** (autoscaling), **assigned stations** (scheduling rules), **walls between kitchens** (NetworkPolicy) and a **single written rulebook** that everything follows (GitOps).

## RBAC: who can do what

**RBAC** (Role-Based Access Control) decides which *subject* may perform which *action* on which *resource*.

| Object | What it is |
|---|---|
| **ServiceAccount** | An identity for a *Pod* (a program, not a person) |
| **Role** | A list of permissions **inside one namespace** ("may `get` and `list` Pods") |
| **ClusterRole** | The same, **cluster-wide** (or for cluster-level objects such as Nodes) |
| **RoleBinding** | Attaches a Role to a user/group/ServiceAccount in a namespace |
| **ClusterRoleBinding** | Attaches a ClusterRole across the whole cluster |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: demo }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: read-pods, namespace: demo }
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Check permissions:

```bash
kubectl auth can-i delete pods -n demo --as=system:serviceaccount:demo:my-app
kubectl auth can-i --list -n demo
```

**Rule of thumb: least privilege.** Give each app its own ServiceAccount with only the verbs it needs. Avoid `cluster-admin` and wildcards (`*`).

## Scheduling: choosing where Pods run

The **scheduler** picks a node for each Pod. You can steer it:

| Tool | Meaning | Example |
|---|---|---|
| **nodeSelector** | "Only nodes with this label" | `disktype: ssd` |
| **Node affinity** | The same, with soft preferences and richer rules | "prefer SSD nodes, require Linux" |
| **Pod affinity / anti-affinity** | "Near" or "away from" other Pods | Spread the 3 replicas across different nodes |
| **Taints** (on a node) | "Keep Pods away unless they tolerate me" | A node reserved for databases |
| **Tolerations** (on a Pod) | "I accept that taint" | The database Pod |
| **topologySpreadConstraints** | Spread Pods evenly across zones/nodes | High availability |

Analogy: a taint is a "staff only" sign on a door; a toleration is the staff badge.

```yaml
spec:
  nodeSelector:
    disktype: ssd
  tolerations:
    - key: dedicated
      operator: Equal
      value: database
      effect: NoSchedule
```

> On a single-node homelab most of this has no visible effect. It becomes essential with several nodes.

**Priority:** a `PriorityClass` lets important Pods preempt (evict) less important ones when the cluster is full.

## Autoscaling

| Scaler | Scales | Based on |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | The **number of Pods** | CPU, memory or custom metrics |
| **VPA** (Vertical Pod Autoscaler) | The **requests/limits** of each Pod | Observed usage (an add-on) |
| **Cluster Autoscaler** | The **number of nodes** | Pods that cannot be scheduled (cloud only) |

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: app }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

The HPA needs **metrics-server** and CPU **requests** on the Pods (utilization is measured *relative to the request*). Without requests, the HPA shows `<unknown>`.

## NetworkPolicy: walls between Pods

By default **every Pod can talk to every other Pod**, in any namespace. A **NetworkPolicy** restricts that, like a firewall written in labels.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-only-from-api, namespace: demo }
spec:
  podSelector:
    matchLabels: { app: db }          # the policy protects these Pods
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: api } # only the API may connect
      ports:
        - port: 5432
```

Once a Pod is selected by a policy, everything not explicitly allowed is blocked. A common pattern is "deny all, then allow what is needed". Policies are enforced by the network plugin (CNI); not every CNI supports them.

## Pod disruption and graceful shutdown

- **PodDisruptionBudget (PDB):** "never let voluntary actions (node drain, upgrades) take down more than 1 Pod of this app" (or "keep at least 2 running").
- **Graceful shutdown:** on delete, Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30 s) and then sends `SIGKILL`. Your app should finish current requests on `SIGTERM`. A `preStop` hook can add a small delay so the Service stops sending traffic first.
- **Drain:** `kubectl drain <node>` moves Pods off a node for maintenance, respecting PDBs; `kubectl cordon <node>` only stops new Pods from arriving.

## Pod security

A Pod can be locked down with a **securityContext**:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities: { drop: ["ALL"] }
```

**Pod Security Admission** enforces profiles per namespace (`privileged`, `baseline`, `restricted`) through namespace labels, rejecting Pods that break them.

## Extending Kubernetes: CRDs, operators and admission

- **CustomResourceDefinition (CRD):** teaches the API a new object kind. After installing the ArgoCD CRDs, `kind: Application` becomes a valid object, like `Deployment`.
- **Operator:** a CRD plus a controller that knows how to run something complex. You write "I want a PostgreSQL cluster with 3 instances" and the operator creates the Pods, storage, backups and failover. Same control-loop idea as in the basics, applied to your own object.
- **Admission controllers / webhooks:** code that can *validate* or *change* an object just before it is saved (for example "reject any image tagged `latest`", or "inject a sidecar").

```bash
kubectl get crd                    # custom kinds installed
kubectl api-resources              # every kind the API knows
kubectl explain deployment.spec.strategy   # built-in documentation for any field
```

## Helm and Kustomize

Two ways to avoid copy-pasting YAML between environments:

| Tool | Idea | Good when |
|---|---|---|
| **Helm** | A *chart* is a package with templates and a `values.yaml`; `helm install` renders and applies it | Installing third-party software (Prometheus, ArgoCD) |
| **Kustomize** | Plain YAML as a *base*, plus *overlays* that patch it (built into `kubectl -k`) | Your own apps with small per-environment differences |

```bash
helm repo add <name> <url> && helm install <release> <chart> -f values.yaml -n <ns>
kubectl apply -k overlays/dev
kubectl kustomize overlays/dev        # preview the final YAML without applying
```

## GitOps

**GitOps** means git is the single source of truth for the cluster. A tool inside the cluster (ArgoCD, Flux) watches the repository and makes the cluster match it.

```
developer ──push──► git repo (manifests) ◄──watches── ArgoCD ──applies──► cluster
                                                         │
                                  detects drift and (optionally) reverts it
```

Benefits: every change has an author, a review and a history; rollback is `git revert`; the cluster can be rebuilt from the repository. Consequence: **changes made only with `kubectl` on a managed object get reverted** by the self-heal feature. Change the manifest instead.

A typical pipeline: code push → CI builds and tags the image → CI updates the image tag in the manifests → ArgoCD syncs → rolling update. See the [tutorial](../tutorials/how-to-deploy-a-frontend-with-argocd-and-a-github-actions-release-pipeline.md).

### Keeping secrets out of git

Since git holds everything, Secrets need special care: commit only `*.example.yaml` with placeholders, or commit them **encrypted** (Sealed Secrets, SOPS), or fetch them from an external store (External Secrets Operator + a vault).

## Observability

| Signal | Question it answers | Typical tool |
|---|---|---|
| **Metrics** | "How much, how fast?" (numbers over time) | Prometheus + Grafana |
| **Logs** | "What happened?" (text events) | Fluent Bit → OpenObserve/Loki |
| **Traces** | "Where did this request spend its time?" | OpenTelemetry |
| **Events** | "What did Kubernetes itself do?" | `kubectl get events` |

See the [tutorial on shipping logs](../tutorials/how-to-ship-application-logs-to-openobserve-with-fluent-bit.md).

## How a request becomes a running Pod (under the hood)

1. `kubectl apply` sends the Deployment to the **API server**, which authenticates you, checks RBAC, runs admission, and stores it in **etcd**.
2. The **Deployment controller** sees it and creates a ReplicaSet.
3. The **ReplicaSet controller** creates the Pod objects (still without a node).
4. The **scheduler** picks a node for each Pod and writes it into the Pod.
5. The **kubelet** on that node sees the Pod assigned to it, asks the container runtime (containerd) to pull the image and start the containers, and runs the probes.
6. The **endpoints** controller adds the Pod to the Service once it is Ready; **kube-proxy** (or the CNI) updates the node's network rules so the Service address reaches it.

Every component only watches the API and reacts. None talks to the others directly. This is why the system recovers by itself after failures.

## Trying it yourself

The hands-on exercises for this level live in a tutorial: [How to practice Kubernetes advanced concepts](../tutorials/how-to-practice-kubernetes-advanced-concepts.md).

## Common confusions

| You might think... | But actually... |
|---|---|
| "A Role grants access cluster-wide." | A Role is namespaced; use a ClusterRole + ClusterRoleBinding for cluster-wide. |
| "HPA works without requests." | Utilization is relative to the CPU request; without it the HPA cannot compute. |
| "Namespaces are walled off by default." | Pods talk across namespaces until a NetworkPolicy says otherwise. |
| "A NetworkPolicy always works." | Only if the CNI enforces it. |
| "I fixed it with `kubectl edit` and it stuck." | With GitOps, self-heal reverts it; change the repository. |
| "PDBs stop all disruptions." | They only affect *voluntary* ones (drain), not crashes or node failures. |
| "An operator is a special kind of Pod." | It is a normal Deployment running a controller that watches a CRD. |

## Summary

- **RBAC** (ServiceAccount, Role, Binding) grants least-privilege access; **securityContext** and Pod Security limit what a Pod can do.
- **Scheduling rules** (affinity, taints, tolerations, spread) place Pods; **HPA/VPA/Cluster Autoscaler** scale them; **PDB** protects availability during maintenance.
- **NetworkPolicy** adds walls (if the CNI supports it).
- **CRDs, operators and admission** extend Kubernetes with the same control-loop idea.
- **Helm/Kustomize** template YAML; **GitOps** makes git the source of truth, so change manifests, not the live cluster.

## References

- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/)
