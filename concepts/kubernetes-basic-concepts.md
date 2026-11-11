# Kubernetes: basic concepts

> **In one sentence:** Kubernetes is a manager that keeps your applications running: you describe what you want ("3 copies of this app, reachable on this address") and it works nonstop to make reality match that description.

- **Last updated:** 2026-11-11
- **Related:** [hands-on tutorial](../tutorials/how-to-practice-kubernetes-basic-concepts.md), [intermediate concepts](kubernetes-intermediate-concepts.md), [advanced concepts](kubernetes-advanced-concepts.md), [DNS](dns.md), [tutorial: limiting old ReplicaSets](../tutorials/how-to-limit-the-old-replicasets-kept-by-a-deployment.md)

## The idea in plain words

Imagine a restaurant kitchen with a **head chef** and many **cooks**. You do not tell each cook what to do minute by minute. You hand the chef the menu: "We need 3 plates of pasta ready at all times." If a cook gets sick, the chef sends another one. If more customers arrive, you change the menu to "6 plates" and the chef organizes it.

Kubernetes works the same way:

- You write down the **desired state** (what you want).
- Kubernetes compares it with the **current state** (what exists).
- If they differ, it acts until they match. This never stops. It is called a **control loop** (or reconciliation loop).

Where the analogy stops: a chef gets tired and forgets. Kubernetes checks again every few seconds, forever, and it does not need you to tell it twice.

**Why it exists:** running one container on one machine is easy. Running dozens of them, restarting the ones that crash, updating them without downtime and giving them stable addresses is hard to do by hand. Kubernetes automates all of that.

## The big picture

```
                 You (kubectl / YAML files)
                           │
                           ▼
        ┌──────────── Control plane ────────────┐
        │  API server  ·  scheduler  ·  etcd    │   ← the "head chef"
        └───────────────────┬───────────────────┘
                            │ "run this Pod on that node"
              ┌─────────────┴─────────────┐
              ▼                           ▼
        ┌─── Node 1 ───┐            ┌─── Node 2 ───┐   ← the "cooks"
        │ kubelet      │            │ kubelet      │
        │ [Pod] [Pod]  │            │ [Pod]        │
        └──────────────┘            └──────────────┘
```

| Part | What it is | Everyday comparison |
|---|---|---|
| **Cluster** | The whole group: control plane + nodes | The whole restaurant |
| **Node** | A machine (physical or virtual) that runs your applications | One cook's workstation |
| **Control plane** | The "brain": decides what runs where and remembers the desired state | The head chef and the order book |
| **API server** | The front door. Everything (you, tools, other parts) talks to the cluster through it | The order window |
| **etcd** | The database where the desired state is stored | The order book |
| **Scheduler** | Picks which node runs each new Pod | The chef assigning orders to cooks |
| **kubelet** | An agent on each node that starts and watches the containers | The cook following orders |

In a homelab with k3s there is usually **one node** that is both control plane and worker. The ideas are the same, just in one machine.

## The building blocks

### Container (and image)

A **container** is an application packed with everything it needs to run (libraries, settings), isolated from the rest of the machine. An **image** is the frozen template a container is created from, like a recipe versus a cooked dish. Images live in a **registry** (Docker Hub, GitHub Container Registry...).

Kubernetes does not run containers directly. It runs them inside Pods.

### Pod

> The smallest thing Kubernetes can run.

A **Pod** is a wrapper around one container (sometimes a few that work closely together). Containers in the same Pod:

- share the same network address (they talk to each other through `localhost`);
- can share disk volumes;
- always run on the same node and live and die together.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
```

The important fact: **Pods are disposable.** If a Pod dies, it is not repaired; it is replaced by a *new* Pod with a *new* name and a *new* address. That is why you almost never create Pods by hand. You create something that manages Pods for you (below).

### ReplicaSet

A **ReplicaSet** has one job: *make sure N identical Pods are running at all times.*

You say "I want 3 replicas of this Pod". If one disappears, the ReplicaSet creates another. If there are too many, it deletes the extras.

```
ReplicaSet (replicas: 3)
   ├── Pod hello-abc12
   ├── Pod hello-def34
   └── Pod hello-ghi56      ← if this one dies, a new one appears
```

ReplicaSets know how to keep Pods alive, but not how to **update** them to a new version. That is the next block's job.

### Deployment

A **Deployment** is the object you will use most. It manages ReplicaSets, and ReplicaSets manage Pods:

```
Deployment  →  ReplicaSet  →  Pods
```

What the Deployment adds:

- **Rolling updates:** when you change the image version, it creates a new ReplicaSet and slowly moves Pods from the old one to the new one, so the app is never fully down.
- **Rollback:** the old ReplicaSet is kept (scaled to zero), so you can go back to the previous version in seconds.
- **Scaling:** change `replicas` and it adjusts.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:                 # the Pod "blueprint"
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: web
          image: nginx:1.27
```

Commands you will use all the time:

```bash
kubectl apply -f deployment.yaml              # create or update
kubectl get deployment hello                  # status
kubectl scale deployment hello --replicas=5   # change the number of Pods
kubectl set image deployment/hello web=nginx:1.28   # new version (rolling update)
kubectl rollout status deployment/hello       # follow the update
kubectl rollout undo deployment/hello         # go back one version
```

### Labels and selectors

A **label** is a sticker on any object: a `key: value` pair such as `app: hello` or `env: prod`. A **selector** is a question like "give me everything with `app: hello`".

This is how objects find each other. The Deployment's `selector` finds its Pods. A Service (next) finds the Pods it should send traffic to. Nothing is connected by name; everything is connected by labels.

```bash
kubectl get pods -l app=hello          # only Pods with that label
kubectl get pods --show-labels         # see the labels
```

> **Pitfall:** a typo in a label means the Service or Deployment silently matches nothing. When "it does not connect", compare the labels first.

### Namespace

A **Namespace** is a folder that groups objects in a cluster. You can have `fintracker-dev`, `messaging` and `monitoring` namespaces, each with its own Deployments and Services, without name clashes. They are also where you apply quotas and permissions.

```bash
kubectl get namespaces
kubectl get pods -n messaging          # -n picks the namespace
kubectl get pods -A                    # all namespaces
```

Names are unique *inside* a namespace, not across the whole cluster.

### Service

Remember: Pods are disposable and each new Pod gets a new address. How can anyone talk to your app if the address keeps changing?

A **Service** is a **stable address and name** placed in front of a group of Pods. It picks its Pods by label and spreads the traffic between them.

Analogy: a company's main phone number. Employees come and go, but the main number stays; the receptionist forwards your call to whoever is available.

```
client ──► Service "hello" (stable name and address)
                │   selector: app=hello
                ├──► Pod hello-abc12
                ├──► Pod hello-def34
                └──► Pod hello-ghi56
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello          # which Pods receive the traffic
  ports:
    - port: 80          # the port of the Service
      targetPort: 80    # the port inside the Pod
```

Inside the cluster, other Pods reach it by name: `http://hello` (same namespace) or `http://hello.<namespace>.svc.cluster.local` (from anywhere). See [DNS inside Kubernetes](dns.md).

### Service types

The Service **type** decides *who* can reach it.

| Type | Who can reach it | Typical use | Everyday comparison |
|---|---|---|---|
| **ClusterIP** (default) | Only things **inside** the cluster | Databases, internal APIs, anything that should not be public | An internal extension number: works only inside the company |
| **NodePort** | Anyone who can reach the **node's IP**, on a high port (30000-32767) | Quick tests, simple setups without a load balancer | A side door on the building, numbered 30080 |
| **LoadBalancer** | Anyone, through its **own external IP** | Exposing an app directly with a real IP (on a homelab, MetalLB hands out the IP) | A dedicated street entrance with its own address |
| **ExternalName** | Pods that ask for it; it just returns another DNS name | Pointing to something outside the cluster (a managed database) | A forwarding note: "we moved, go to this address" |

They build on each other:

```
LoadBalancer  ⊃  NodePort  ⊃  ClusterIP
```

A LoadBalancer Service also gets a NodePort and a ClusterIP. A NodePort Service also gets a ClusterIP.

```yaml
# NodePort example
spec:
  type: NodePort
  selector: { app: hello }
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080     # optional; Kubernetes picks one if omitted
```

Reach it with `http://<NODE_IP>:30080`.

```yaml
# LoadBalancer example
spec:
  type: LoadBalancer
  selector: { app: hello }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl get svc hello
# NAME    TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)
# hello   LoadBalancer   <CLUSTER_IP>    <LOADBALANCER_IP>  80:3xxxx/TCP
```

If `EXTERNAL-IP` stays `<pending>`, nothing is giving out IPs (on a homelab, install MetalLB).

**Which one to pick?** Start with **ClusterIP** for everything. Expose to the outside only what must be reachable. For HTTP apps, prefer an [Ingress](kubernetes-intermediate-concepts.md#ingress) over many LoadBalancers.

### The three ports (a classic confusion)

| Field | Meaning |
|---|---|
| `port` | The port the **Service** listens on |
| `targetPort` | The port **inside the Pod** where the app listens |
| `nodePort` | The port opened on the **node** (NodePort/LoadBalancer only) |

## Trying it yourself

The hands-on exercises for this level live in a tutorial: [How to practice Kubernetes basic concepts](../tutorials/how-to-practice-kubernetes-basic-concepts.md).

## Common confusions

| You might think... | But actually... |
|---|---|
| "A Pod is a container." | A Pod *wraps* one or more containers. It is the unit Kubernetes schedules. |
| "I should create Pods directly." | Create a Deployment. A bare Pod is not replaced if it dies or its node fails. |
| "The Pod's IP is stable." | It changes every time the Pod is recreated. Use a Service. |
| "NodePort and LoadBalancer are totally different." | LoadBalancer is NodePort plus an external IP. |
| "`kubectl delete pod` removes my app." | The ReplicaSet creates a new one. Delete the Deployment to remove the app. |
| "Namespaces isolate the network." | By default, Pods in different namespaces can still talk. Isolation needs NetworkPolicies. |

## Summary

- Kubernetes keeps **desired state = current state**, in a loop.
- **Pod** runs containers; **ReplicaSet** keeps N Pods; **Deployment** manages ReplicaSets and gives rolling updates and rollback.
- **Labels/selectors** connect objects; **Namespaces** group them.
- **Service** gives Pods a stable address: **ClusterIP** (inside), **NodePort** (node port), **LoadBalancer** (own external IP).

## References

- [Kubernetes concepts](https://kubernetes.io/docs/concepts/)
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
