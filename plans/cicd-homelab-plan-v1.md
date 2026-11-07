# Homelab CI/CD Architecture Plan: Expense Tracker

## 1. Architectural Overview
This document outlines the implementation plan for a fully automated GitOps CI/CD pipeline running on a local Debian-based K3s server. The architecture uses a single-cluster approach with strict logical isolation via Kubernetes Namespaces and Resource Quotas. This ensures that CI build processes (which can cause CPU/Memory spikes) do not impact the live application workloads, effectively utilizing the host system's 20GB memory capacity.

Implementing this standard GitOps structure provides the robust infrastructure orchestration and automation expected in senior-level engineering environments.

### Core Components
* **Source Control:** GitHub (hosts application code and Kubernetes manifests).
* **CI Runner:** GitHub Actions Runner Controller (ARC) deployed inside the K3s cluster.
* **Container Registry:** Docker Hub.
* **CD Controller:** ArgoCD deployed inside the K3s cluster.
* **Target Workloads:** Expense Tracker application (FastAPI backend and Vue.js 3 frontend).

---

## 2. Cluster Topology & Logical Isolation

The K3s cluster will be divided into two primary namespaces to separate the management plane from the application plane.

### 2.1. Namespaces
1.  `cicd-system`: Hosts ArgoCD and the Actions Runner Controller (ARC).
2.  `fintracker-dev`: Hosts the deployed Expense Tracker services (FastAPI backend, Vue.js frontend. It's already running).

### 2.2. Resource Quotas
To prevent the self-hosted GitHub runners from starving the application of resources during Docker builds, a `ResourceQuota` must be applied to the `cicd-system` namespace.

**Constraint Definition** (`manifests/ci-cd/cicd-system-resource-quota.yaml`):
* Maximum Memory (limits) for `cicd-system`: 4Gi
* Maximum CPU (limits) for `cicd-system`: 3 Cores
* Maximum CPU (requests) for `cicd-system`: 1 Core

The CPU limit was raised from 2 to 3 cores: ArgoCD alone needs 2.2 cores of limits, and 2 cores left the `argocd-application-controller` blocked by the quota. See `troubleshooting/2026-11-07-argocd-resource-quota.md`.

**Sizing rule:** every container in `cicd-system` must declare explicit `resources`. Containers that do not declare them get the LimitRange default (500m CPU / 512Mi), which counts against the quota. Keep headroom for rolling updates, when the old and new pods exist at the same time. Re-check the totals before adding any new workload to this namespace.

---

## 3. Implementation Phases

### Phase 1: Namespace and Quota Provisioning
1.  Create the `cicd-system` namespace.
2.  Define and apply the `ResourceQuota` and `LimitRange` manifests to the `cicd-system` namespace.

### Phase 2: Continuous Deployment (ArgoCD) Setup
ArgoCD is installed **with Kustomize, not Helm**, from the official installation manifest. The files live in `manifests/ci-cd/`:

| File | Purpose |
|---|---|
| `kustomization.yaml` | Pulls the official manifest (pinned to `v3.3.8`), sets the `cicd-system` namespace, and includes the namespace, quota and LimitRange |
| `argocd-resources.yaml` | Explicit CPU/memory for every container, including init containers, so the pods fit the quota |
| `argocd-cmd-params.yaml` | Runs the server with `--insecure` (Traefik terminates TLS) |
| `namespace.yaml`, `cicd-system-resource-quota.yaml`, `cicd-system-limit-range.yaml` | Namespace, quota and LimitRange (Phase 1) |

The kustomization also patches the ClusterRoleBinding subjects: the official manifest hard-codes the `argocd` namespace there, and without this patch ArgoCD has no permissions in the cluster.

1.  Preview the rendered manifests: `kubectl kustomize manifests/ci-cd/`
2.  Apply them: `kubectl apply -k manifests/ci-cd/ --server-side --force-conflicts`. Server-side apply is required, because the ArgoCD CRDs are too large for client-side apply. If the first run fails with `no matches for kind "AppProject"`, run it again once the CRDs are registered.
3.  Verify that all 7 pods are `Running` and that the services exist: `kubectl get pods,svc -n cicd-system`. If a pod is missing, check `kubectl get events -n cicd-system` for quota errors.
4.  Expose the ArgoCD API/UI. For now use port-forwarding: `kubectl port-forward -n cicd-system svc/argocd-server 8080:80`. An Ingress through Traefik is a later step.
5.  Retrieve the initial `admin` password from the cluster, log in, change it, and then delete the `argocd-initial-admin-secret` secret. Never commit the password to the repository.
    ```bash
    kubectl -n cicd-system get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
    ```

To upgrade ArgoCD, change the pinned version in `kustomization.yaml`, preview, and apply again. Check that the container names in `argocd-resources.yaml` still match the new manifest.

### Phase 3: Continuous Integration (ARC) Setup
1.  Generate a GitHub Personal Access Token (PAT) or set up a GitHub App with permissions to manage runners for the repository.
2.  Store this token as a Kubernetes Secret in the `cicd-system` namespace.
3.  Install the Actions Runner Controller (ARC) via Helm into `cicd-system`.
4.  Apply a `RunnerDeployment` manifest configured to listen to the target GitHub repository and scale the runner pods based on webhook events.

### Phase 4: Container Registry Configuration
1.  Create a standard Access Token in Docker Hub.
2.  Add `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` as Repository Secrets in the GitHub settings for the Expense Tracker repository.

### Phase 5: CI Pipeline Implementation (.github/workflows)
Create a workflow file (e.g., `.github/workflows/deploy.yml`) containing the following sequence of operations:
1.  **Trigger:** Execute on pushes to the `main` branch.
2.  **Environment:** Set `runs-on: self-hosted` to target the ARC runner in the local K3s cluster.
3.  **Authentication:** Use the `docker/login-action` with the configured Docker Hub secrets.
4.  **Build & Push:** Use `docker/build-push-action` to build the FastAPI and Vue.js Docker images and push them to Docker Hub. Tag the images using the short Git commit SHA (`${{ github.sha }}`).
5.  **Manifest Update:** 
    * Check out the repository containing the Kubernetes deployment manifests.
    * Use `sed` or a tool like `yq` to programmatically update the image tag in the deployment YAML files to the new commit SHA.
    * Commit and push the updated manifest file back to the main branch.

### Phase 6: GitOps Integration
1.  Within ArgoCD, configure a new `Application` resource.
2.  Set the source repository to the Git path where the Kubernetes manifests are stored.
3.  Set the destination to the local K3s cluster and the `fintracker-dev` namespace.
4.  Enable the Automated Sync policy with Self-Heal and Prune options active.

---

## 4. Execution Directives for AI Assistant
When implementing this plan, generate the required YAML manifests and CLI commands iteratively:
1.  Begin by generating the Namespace and ResourceQuota manifests.
2.  Provide the Helm values and installation commands for ARC. ArgoCD is installed with Kustomize (`manifests/ci-cd/`), not Helm: keep it that way, and do not run `helm install` for ArgoCD.
3.  Draft the exact GitHub Actions workflow YAML.
4.  Ensure all Kubernetes manifests specify the correct namespaces (`cicd-system` or `fintracker-dev`).