# How to practice Kubernetes basic concepts

> **Summary:** a hands-on lab in a throwaway namespace where you create a Deployment, break it on purpose, watch Kubernetes repair it, update and roll back a version, and expose it with ClusterIP and NodePort Services. At the end you can explain Pod, ReplicaSet, Deployment, labels, Namespace and Service from experience.

- **Last updated:** 2026-11-11
- **Components:** any Kubernetes cluster (written for k3s), `kubectl`, public images `nginx` and `busybox`
- **Level:** beginner
- **Theory:** [Kubernetes: basic concepts](../concepts/kubernetes-basic-concepts.md)

## Context

Reading about Pods and ReplicaSets helps, but the ideas only stick when you see a Pod get replaced a second after you delete it. This lab is safe: everything happens in one Namespace that you delete at the end.

## Prerequisites

- A working cluster and `kubectl` pointing at it (`kubectl get nodes` shows a `Ready` node).
- Permission to create Namespaces.
- Two terminals. In the second one you will run `kubectl get ... -w` (watch) to see changes live.

## Overview

1. Create a Namespace.
2. Create a Deployment and inspect the chain Deployment → ReplicaSet → Pod.
3. Delete a Pod and watch the self-healing.
4. Scale up and down.
5. Do a rolling update and a rollback.
6. Expose the app with a ClusterIP Service and test it from inside.
7. Break the Service with a wrong label, then fix it.
8. Expose it with NodePort.
9. Clean up.

## Step by step

### 1. Create the Namespace

A Namespace is a folder for objects. Using a dedicated one lets you delete everything with one command later.

```bash
kubectl create namespace practice-basic
kubectl config set-context --current --namespace=practice-basic   # so you can omit -n below
```

**How to check:**

```bash
kubectl get namespace practice-basic
```

Expected: `STATUS Active`.

### 2. Create a Deployment and inspect the chain

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: web
          image: nginx:1.26
          ports:
            - containerPort: 80
EOF
```

**Why:** `replicas: 2` asks for two Pods. `selector` + `template.metadata.labels` is how the Deployment recognizes its own Pods; they must match.

**How to check:**

```bash
kubectl get deployment,replicaset,pods
```

Expected: one Deployment `hello` (`2/2` ready), one ReplicaSet `hello-<hash>`, two Pods `hello-<hash>-<random>`.

**Look closely:** the Pod name contains the ReplicaSet name, which contains the Deployment name. Follow the chain with:

```bash
kubectl describe pod -l app=hello | grep -E 'Name:|Controlled By|Node:|IP:'
```

`Controlled By: ReplicaSet/hello-<hash>` proves who owns the Pod. Notice each Pod has its own IP.

### 3. Delete a Pod and watch the self-healing

In the **second terminal**:

```bash
kubectl get pods -w
```

In the first one:

```bash
kubectl delete pod -l app=hello --wait=false     # deletes both Pods
```

**What to observe:** the old Pods go `Terminating` while two new ones appear with different names and different IPs. The ReplicaSet saw "current 0, desired 2" and acted.

**What you learned:** Pods are disposable; the desired state lives in the Deployment.

### 4. Scale up and down

```bash
kubectl scale deployment hello --replicas=5
kubectl get pods
kubectl scale deployment hello --replicas=2
```

**What to observe:** three Pods are created and then three are deleted. The ReplicaSet name did not change, because the Pod template did not change.

### 5. Rolling update and rollback

Change the image: this changes the Pod template, so a **new** ReplicaSet is created.

```bash
kubectl set image deployment/hello web=nginx:1.27
kubectl rollout status deployment/hello
kubectl get replicaset
```

**Expected:** two ReplicaSets: the new one with `DESIRED 2` and the old one with `DESIRED 0`. The old one is kept on purpose: it is your rollback.

```bash
kubectl rollout history deployment/hello
kubectl rollout undo deployment/hello
kubectl get replicaset
kubectl describe deployment hello | grep Image
```

**Expected:** the image is `nginx:1.26` again, and the *old* ReplicaSet is the active one. Rollback re-used it, it did not build anything new.

> To see how many old ReplicaSets are kept and how to limit them, read [How to limit the old ReplicaSets kept by a Deployment](how-to-limit-the-old-replicasets-kept-by-a-deployment.md).

### 6. Expose the app with a ClusterIP Service

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
EOF
```

`type` was omitted, so it is **ClusterIP**: reachable only from inside the cluster.

**How to check:**

```bash
kubectl get service hello
kubectl get endpoints hello       # the Pod IPs behind the Service
```

Expected: a `CLUSTER-IP` and two entries in `ENDPOINTS`, matching the Pod IPs from step 2.

Test it from a temporary Pod inside the cluster:

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://hello
```

Expected: the nginx welcome HTML. You used the Service **name**, resolved by the cluster DNS (see [DNS](../concepts/dns.md)). The long form also works: `http://hello.practice-basic.svc.cluster.local`.

Now prove the Service is stable while Pods change:

```bash
kubectl delete pod -l app=hello --wait=false
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://hello | head -3
```

The Pods are new, the address is the same, and it still answers.

### 7. Break the Service with a wrong label, then fix it

Labels are the glue. Break it:

```bash
kubectl patch service hello -p '{"spec":{"selector":{"app":"helo"}}}'    # typo on purpose
kubectl get endpoints hello
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- -T 3 http://hello
```

**Expected:** `ENDPOINTS` is `<none>` and `wget` times out. The Service exists but matches no Pods. This is the most common "it does not connect" cause.

Fix it:

```bash
kubectl patch service hello -p '{"spec":{"selector":{"app":"hello"}}}'
kubectl get endpoints hello
```

### 8. Expose it with NodePort

```bash
kubectl patch service hello -p '{"spec":{"type":"NodePort"}}'
kubectl get service hello
```

**Expected:** `TYPE NodePort` and `PORT(S) 80:3xxxx/TCP`. The number after the colon is the port opened on **every node**.

From a machine that can reach the node:

```bash
curl http://<NODE_IP>:<NODEPORT>
```

**What you learned:** the same Service gained a node port and still has its ClusterIP (NodePort ⊃ ClusterIP). If you also want a dedicated external IP, the type would be `LoadBalancer` (on a homelab this needs something like MetalLB).

### 9. Clean up

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace practice-basic
```

Deleting the Namespace deletes everything inside it.

## Technical decisions

### Using a dedicated, throwaway Namespace

- **Decision:** run the whole lab in `practice-basic` and delete it at the end.
- **Why:** one command removes everything, and nothing touches real workloads.
- **Alternatives considered:** the `default` namespace (leaves clutter and can clash with existing names).
- **Trade-offs / risks:** changing the current context's namespace affects later commands; step 9 resets it.

### Pinning image versions

- **Decision:** `nginx:1.26` and `nginx:1.27`, never `latest`.
- **Why:** the rolling update only has something to do when the tag changes, and pinned tags make the lab reproducible.
- **Alternatives considered:** `latest` (a rollout would not trigger and rollback would be meaningless).
- **Trade-offs / risks:** versions get old; use any two different tags that exist.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| Pods stay `ImagePullBackOff` | Wrong tag or no internet from the node | `kubectl describe pod` → read the Events; fix the tag |
| `ENDPOINTS` is `<none>` | Service selector does not match the Pod labels | Compare `kubectl get pods --show-labels` with `kubectl get svc hello -o yaml` |
| `kubectl run` hangs or fails | A leftover `tmp` Pod | `kubectl delete pod tmp` |
| Cannot reach the NodePort | Firewall between you and the node, or wrong port | Check `kubectl get svc` for the real port; test from the node first |

## How to undo

```bash
kubectl delete namespace practice-basic
```

## Exercises to go further

1. Create a second Deployment `hello2` with label `app: hello2` and make the Service send traffic to **both** apps. (Hint: which label do both Pods share? Add one.)
2. Set `replicas: 0` on the Deployment. What happens to the Service's endpoints? What happens to the ReplicaSet?
3. Create a bare Pod (`kind: Pod`) and delete it. Why is it not replaced?
4. Change the Service type to `LoadBalancer` and read `EXTERNAL-IP`. What does `<pending>` tell you?

## References

- [Kubernetes: basic concepts](../concepts/kubernetes-basic-concepts.md)
- [Kubernetes: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes: Service](https://kubernetes.io/docs/concepts/services-networking/service/)
