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
The namespace, ResourceQuota and LimitRange are part of the ArgoCD kustomization
(`manifests/ci-cd/kustomization.yaml`), so Phase 2 creates them too. To apply them on their own:
```bash
kubectl apply -f manifests/ci-cd/namespace.yaml
kubectl apply -f manifests/ci-cd/cicd-system-resource-quota.yaml
kubectl apply -f manifests/ci-cd/cicd-system-limit-range.yaml

# Verify
kubectl get namespace cicd-system
kubectl describe resourcequota cicd-system-quota -n cicd-system
kubectl describe limitrange cicd-system-limits -n cicd-system
```

### Phase 2 — ArgoCD Installation (Kustomize, no Helm)
ArgoCD v3.3.8 (official manifest, pinned) is installed into `cicd-system` with Kustomize.
The kustomization adds the namespace/quota/limit range, sets CPU/memory on every container
(`argocd-resources.yaml`), runs the server without TLS (`argocd-cmd-params.yaml`), fixes the
ClusterRoleBinding subjects to `cicd-system` and adds the Traefik ingress (`argocd-ingress.yaml`).

```bash
# 1. Preview what will be created (no changes to the cluster)
kubectl kustomize manifests/ci-cd/ | less
kubectl apply -k manifests/ci-cd/ --server-side --dry-run=server

# 2. (Optional) In a second terminal, watch the pods come up
kubectl get pods -n cicd-system -w
kubectl get events -n cicd-system -w     # quota errors only show up here

# 3. Install. --server-side is required: the CRDs are too large for client-side apply.
#    Do not interrupt it halfway (see troubleshooting/2026-11-07-argocd-resource-quota.md).
kubectl apply -k manifests/ci-cd/ --server-side --force-conflicts

# 4. Wait until everything is ready
kubectl rollout status statefulset/argocd-application-controller -n cicd-system
kubectl wait --for=condition=available deployment --all -n cicd-system --timeout=300s

# 5. Verify: 7 pods Running, services and ingress present, quota not exceeded
kubectl get all -n cicd-system
kubectl get ingress -n cicd-system
kubectl describe resourcequota cicd-system-quota -n cicd-system

# 6. Add argocd.local to /etc/hosts on the client machine (Traefik LoadBalancer IP)
kubectl get svc -n kube-system traefik     # EXTERNAL-IP, currently 192.168.18.2
#    /etc/hosts line:
#    192.168.18.2  argocd.local

# 7. Retrieve the initial admin password
kubectl get secret argocd-initial-admin-secret \
  -n cicd-system \
  -o jsonpath="{.data.password}" | base64 --decode && echo

# ArgoCD UI is now available at http://argocd.local
# Login: admin / <password from step 7>

# 8. (Recommended) Change the admin password after first login (User Info > Update Password,
#    or `argocd account update-password`), then delete the bootstrap secret
kubectl delete secret argocd-initial-admin-secret -n cicd-system
```

#### Alternative: Port-Forward (no /etc/hosts change needed)
```bash
kubectl port-forward svc/argocd-server -n cicd-system 8080:80
# UI: http://localhost:8080
```

#### Upgrade
Change the version in the `install.yaml` URL in `manifests/ci-cd/kustomization.yaml`, check
that the container names in `argocd-resources.yaml` still match the new manifest, then:
```bash
kubectl kustomize manifests/ci-cd/ > /dev/null    # must build without errors
kubectl apply -k manifests/ci-cd/ --server-side --force-conflicts
```

#### Uninstall
```bash
# Removes everything in the kustomization, including the namespace, CRDs and cluster roles.
# Delete the Applications first so their finalizers do not block the CRD deletion.
kubectl delete applications --all -n cicd-system
kubectl delete -k manifests/ci-cd/
```
