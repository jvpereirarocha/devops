# K3s Cluster Operations Reference

This guide summarizes common commands for managing your K3s homelab cluster.

## Service Management
Commands to manage the K3s systemd service on the host.
- **Check Status:** `sudo systemctl status k3s`
- **Start Service:** `sudo systemctl start k3s`
- **Stop Service:** `sudo systemctl stop k3s`
- **Restart Service:** `sudo systemctl restart k3s`
- **View Logs:** `sudo journalctl -u k3s -f`

## Built-in Tooling
K3s bundles several tools to avoid external dependencies.
- **Kubectl:** `k3s kubectl get nodes` (Usually aliased to `kubectl`)
- **CRICTL (Container Runtime):** `sudo k3s crictl ps`
- **CTR (Containerd):** `sudo k3s ctr containers ls`

## Cluster Maintenance
- **Check Host Requirements:** `k3s check-config`
- **Etcd Snapshots:**
  - Save: `k3s etcd-snapshot save`
  - List: `k3s etcd-snapshot list`
- **Certificate Rotation:** `sudo k3s certificate rotate`

## Node Operations
- **Get Agent Join Token:** `sudo cat /var/lib/rancher/k3s/server/node-token`
- **Node Health:** `kubectl get nodes -o wide`
- **Describe Node:** `kubectl describe node <node-name>`

## Debugging
- **Check K3s Config:** `cat /etc/rancher/k3s/config.yaml`
- **Kubeconfig Location:** `/etc/rancher/k3s/k3s.yaml`

---

## CI/CD Pipeline Setup

### Phase 1 — Namespace & Quotas
```bash
kubectl apply -f manifests/ci-cd/namespace.yaml
kubectl apply -f manifests/ci-cd/cicd-system-resource-quota.yaml
kubectl apply -f manifests/ci-cd/cicd-system-limit-range.yaml

# Verify
kubectl get namespace cicd-system
kubectl describe resourcequota cicd-system-quota -n cicd-system
kubectl describe limitrange cicd-system-limits -n cicd-system
```

### Phase 2 — ArgoCD Installation
```bash
# 1. Add the Argo Helm repo (once)
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# 2. Install ArgoCD into cicd-system
helm install argocd argo/argo-cd \
  --namespace cicd-system \
  --values manifests/ci-cd/argocd-values.yaml \
  --wait

# 3. Apply the Traefik ingress
kubectl apply -f manifests/ci-cd/argocd-ingress.yaml

# 4. Add argocd.local to /etc/hosts (point to the Traefik LoadBalancer IP)
#    Find the Traefik external IP first:
kubectl get svc -n kube-system traefik
#    Then add this line to /etc/hosts on the host machine:
#    <TRAEFIK_EXTERNAL_IP>  argocd.local

# 5. Retrieve the initial admin password
kubectl get secret argocd-initial-admin-secret \
  -n cicd-system \
  -o jsonpath="{.data.password}" | base64 --decode && echo

# ArgoCD UI is now available at http://argocd.local
# Login: admin / <password from step 5>

# 6. (Recommended) Change the admin password after first login, then delete the bootstrap secret
kubectl delete secret argocd-initial-admin-secret -n cicd-system
```

#### Alternative: Port-Forward (no /etc/hosts change needed)
```bash
kubectl port-forward svc/argocd-server -n cicd-system 8080:80
# UI: http://localhost:8080
```

#### Upgrade / Uninstall
```bash
helm upgrade argocd argo/argo-cd -n cicd-system -f manifests/ci-cd/argocd-values.yaml
helm uninstall argocd -n cicd-system
```
