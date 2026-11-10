# How to deploy a frontend with ArgoCD and a GitHub Actions release pipeline

> **Summary:** move a frontend that was deployed by hand (`kubectl apply` / `kubectl set image`)
> to GitOps. ArgoCD syncs its manifests from the frontend repo, and a GitHub Actions pipeline
> runs the tests, picks the next version with semantic-release, pushes the image to Docker Hub
> and only then bumps the image tag in git. The frontend shares its namespace with the API, which
> is already managed by another ArgoCD Application.

- **Last updated:** 2026-11-10
- **Components:** k3s v1.34, ArgoCD v3.3.8, Kustomize (built into `kubectl`), GitHub Actions
  (GitHub-hosted runners), semantic-release 24, Docker Hub, nginx (serving a Vite/Vue build)
- **Level:** intermediate

## Context

The app has three workloads in the same namespace (`fintracker-dev`):

| Workload | How it was deployed before |
|---|---|
| API (`fintracker-deployment`) | ArgoCD Application `fintracker`, image tag bumped in git by CI |
| Frontend (`front-fintracker-deployment`) | by hand: `kubectl apply`, then `kubectl set image` |
| Telegram bot | by hand (out of scope here) |

Deploying by hand has two problems:

1. **Git does not describe the cluster.** The manifest in the frontend repo said
   `image: front-fintracker:latest`, while the cluster was running `front-fintracker:3.0`
   (someone ran `kubectl set image` and never updated the file). This is called **drift**.
2. **Every release is manual**: build, push, update the Deployment, hope you did not forget a step.

The goal is the same flow the API already has: merge to `main` → CI releases a version → ArgoCD
deploys it.

## Prerequisites

- A k3s (or any Kubernetes) cluster with ArgoCD installed and able to reach GitHub (see
  [How to deploy ArgoCD on the Kubernetes cluster](how-to-deploy-argocd-on-the-kubernetes-cluster.md)).
- The frontend already running in the cluster, with its manifests (Deployment, Service,
  ConfigMap) in a `k8s/` folder of the frontend repo.
- A **public** GitHub repo for the frontend (a private one needs a repository credential in
  ArgoCD, not covered here).
- A Docker Hub account and a repository for the image.
- `kubectl` and `git` on your machine; Docker is optional, to test the build locally.

## Overview

```
git push to main (frontend repo)
   │
   ▼
GitHub Actions ── test job:    npm ci → oxlint → vitest → vite build
   │
   └──────────── release job:  semantic-release --dry-run  → next version X.Y.Z
                               docker build + push         → image:X.Y.Z
                               semantic-release            → commit "newTag: X.Y.Z", tag vX.Y.Z
   │
   ▼
ArgoCD (polls git every ~3 min) ── sees the new newTag ── syncs the Deployment ── rolling update
```

Steps:

1. Make the manifests reproducible: pin the image tag with Kustomize.
2. Create a separate ArgoCD Application for the frontend.
3. Let ArgoCD adopt the resources that already exist.
4. Add the CI pipeline (tests, versioning, image, tag bump).
5. Create the base git tag and the Docker Hub secrets, then run the first release.
6. Keep only one old ReplicaSet.

## Step by step

### 1. Pin the image tag with Kustomize

**Kustomize** is a tool built into `kubectl` (`kubectl apply -k`, `kubectl kustomize`) that builds
the final YAML from plain manifests plus a `kustomization.yaml`. The `images` field rewrites the
image of every container that uses a given name, so the version lives in one line that CI can
change.

Create `k8s/kustomization.yaml` in the frontend repo:

```yaml
# k8s/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - configmap.yaml
  - deployment.yaml
  - service.yaml

images:
  - name: <dockerhub-user>/front-fintracker
    # Keep it quoted so YAML reads it as a string (3.0 would become a number).
    newTag: "3.0"
```

In `k8s/deployment.yaml`, drop the tag from the image and stop pulling on every start:

```yaml
      containers:
        - name: front-fintracker
          image: <dockerhub-user>/front-fintracker  # tag pinned in kustomization.yaml
          imagePullPolicy: IfNotPresent
```

**Why:**

- **The tag must be the one running today** (`3.0` here, read from the cluster), not `latest`.
  The first ArgoCD sync applies whatever git says. With `latest`, it would replace the running
  version with an unknown one.
- **`latest` breaks GitOps.** ArgoCD compares git with the cluster. If the manifest always says
  `latest`, a new image never changes the manifest, so ArgoCD has nothing to sync.
- **`IfNotPresent`** is enough with immutable tags (each version has its own tag). `Always` is only
  needed for moving tags like `latest`.
- The **Namespace is not declared** here. It already belongs to the API's Application (more in
  [Technical decisions](#the-namespace-belongs-to-one-application-only)).

**How to check:** find the running image, then compare git with the cluster:

```bash
kubectl -n fintracker-dev get deploy front-fintracker-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl kustomize k8s/ | kubectl diff -f - | grep -E '^[+-] '
```

Expected output: the running image is `<dockerhub-user>/front-fintracker:3.0`, and the diff shows
only the fields you meant to change (here, `imagePullPolicy: Always` → `IfNotPresent`). If the
diff shows an image change, the tag in `kustomization.yaml` is wrong.

Commit and push the `k8s/` folder: ArgoCD reads from git, not from your machine.

### 2. Create a separate ArgoCD Application

An **Application** is the ArgoCD object that says "keep this path of this git repo applied to this
namespace". Keep the Application manifests versioned in the infra repo (here
`manifests/ci-cd/applications/`), so the ArgoCD setup can be rebuilt from git too.

```yaml
# manifests/ci-cd/applications/front-fintracker.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: front-fintracker
  namespace: cicd-system          # the namespace where ArgoCD runs
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/<frontend-repo>
    targetRevision: main
    path: k8s                     # ArgoCD detects kustomization.yaml and runs Kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: fintracker-dev     # same namespace as the API
  syncPolicy:
    automated:
      enabled: true
      selfHeal: true              # changes made only in the cluster are reverted
```

If the API's Application was created through the UI or the CLI, export it into the same folder
too (`kubectl -n cicd-system get application <name> -o yaml`, keep only `metadata.name`,
`metadata.namespace` and `spec`).

```bash
kubectl apply -f manifests/ci-cd/applications/
```

**Why `-f` and not `-k` here:** these files are complete manifests, not patches. Keep them out
of the ArgoCD install kustomization, so upgrading ArgoCD never touches the Applications.

### 3. Let ArgoCD adopt the existing resources

You do not need to delete anything first. When the Application syncs, ArgoCD finds the
Deployment, Service and ConfigMap that already exist with the same names, applies the git
version over them and adds its tracking annotation (`argocd.argoproj.io/tracking-id`). From then
on, it owns them.

**How to check:**

```bash
kubectl -n cicd-system get applications
kubectl -n cicd-system get application front-fintracker \
  -o jsonpath='{range .status.resources[*]}{.kind}/{.name} {.status}{"\n"}{end}'
kubectl -n fintracker-dev get pods -l app=front-fintracker
curl -s -o /dev/null -w "%{http_code}\n" http://<LOADBALANCER_IP>/
```

Expected output: both Applications are `Synced` and `Healthy`. The three frontend resources show
up as `Synced`. The pod is `Running`, and the page answers `200`. The only change in step 1 was
`imagePullPolicy`, which is part of the pod template, so the Deployment did a rolling update with
the same image: the new pod started before the old one stopped.

### 4. Add the CI pipeline

The pipeline has two jobs.

**`test`** runs on every push and pull request: `npm ci`, the linter, the unit tests and the
production build. A broken build never gets released.

**`release`** runs only on pushes to `main`. **semantic-release** reads the commit messages since
the last version tag and decides the next version with the
[Conventional Commits](https://www.conventionalcommits.org/) rules:

- `fix:` makes a patch release (3.1.0 → 3.1.1);
- `feat:` makes a minor release (3.1.0 → 3.2.0);
- `BREAKING CHANGE` makes a major release;
- `chore:`, `ci:`, `docs:` and similar make no release.

The order of the steps in this job is the key decision of the tutorial:

1. `semantic-release --dry-run`: compute the next version without changing anything;
2. build and push `image:X.Y.Z` to Docker Hub;
3. `semantic-release` for real: bump `newTag` in `k8s/kustomization.yaml`, commit it, create the
   `vX.Y.Z` tag and the GitHub release.

ArgoCD deploys as soon as `newTag` changes in git. If the tag were bumped **before** the image
existed, ArgoCD could sync first and the pod would fail with `ImagePullBackOff` until the push
finished. Pushing the image first makes that impossible.

The semantic-release config (`.releaserc.json` at the repo root):

```json
{
  "branches": ["main"],
  "tagFormat": "v${version}",
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
      "@semantic-release/exec",
      {
        "verifyReleaseCmd": "echo ${nextRelease.version} > .next-version",
        "prepareCmd": "sed -i 's/newTag: \".*\"/newTag: \"${nextRelease.version}\"/' k8s/kustomization.yaml"
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["k8s/kustomization.yaml"],
        "message": "chore(release): ${nextRelease.version} [skip ci]"
      }
    ],
    "@semantic-release/github"
  ]
}
```

- `verifyReleaseCmd` **also runs in dry-run mode**. That is how the dry run hands the version to
  the next steps (the `.next-version` file, which goes in `.gitignore`).
- `prepareCmd` only runs in the real release, and changes the single `newTag` line.
- `@semantic-release/git` commits that file. The commit is pushed with the workflow's
  `GITHUB_TOKEN`, and GitHub does not start new workflow runs for pushes made with that token,
  so there is no release loop (`[skip ci]` is an extra guard).

The workflow, `.github/workflows/ci.yaml` (trimmed):

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_IMAGE: <dockerhub-user>/front-fintracker
  NODE_VERSION: "22"

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: npm }
      - run: npm ci
      - run: npx oxlint .
      - run: npx vitest run
      - run: npm run build

  release:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push'
    concurrency: { group: "release-${{ github.ref_name }}", cancel-in-progress: false }
    permissions: { contents: write, issues: write, pull-requests: write }
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      SEMREL: >-
        npx --yes -p semantic-release@24.2.3 -p @semantic-release/exec@6.0.3
        -p @semantic-release/git@10.0.1 semantic-release
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # semantic-release needs the full history and tags
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}" }
      - id: next
        run: |
          $SEMREL --dry-run
          if [ -f .next-version ]; then
            echo "version=$(cat .next-version)" >> "$GITHUB_OUTPUT"; rm .next-version
          fi
      - if: steps.next.outputs.version != ''
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - if: steps.next.outputs.version != ''
        uses: docker/setup-buildx-action@v3
      - if: steps.next.outputs.version != ''
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.DOCKER_IMAGE }}:${{ steps.next.outputs.version }}
            ${{ env.DOCKER_IMAGE }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - if: steps.next.outputs.version != ''
        run: $SEMREL
```

The full file is `.github/workflows/ci.yaml` in the frontend repo.

**Before pushing, run the same checks locally** in a clean container, so the pipeline does not
start out red:

```bash
docker run --rm -v "$PWD":/src:ro node:22-alpine sh -c \
  'cp -r /src /app && cd /app && rm -rf node_modules && npm ci && npx oxlint . && npx vitest run && npm run build'
```

In this repo, ESLint failed on errors that were already in the code (`vue/no-mutating-props`),
so the pipeline runs only oxlint for now. Add `npx eslint .` to the `test` job once those are
fixed.

### 5. Create the base tag, the secrets and run the first release

**The base tag.** semantic-release starts from the last `vX.Y.Z` tag. With no tag at all, it
would release `1.0.0`, which is **lower** than the `3.0` already deployed. Tag the commit that
matches the running version:

```bash
git tag -a v3.0.0 <commit> -m "3.0.0: version running in the cluster when CI was added"
git push origin v3.0.0
```

You can check the next version locally without a GitHub token, by running only the commit
analyzer:

```bash
docker run --rm -v "$PWD":/src:ro node:22-alpine sh -c 'apk add -q git; cp -r /src /app && cd /app &&
  git config --global --add safe.directory "*" &&
  npx --yes -p semantic-release@24.2.3 semantic-release --dry-run --no-ci \
    --plugins @semantic-release/commit-analyzer --repository-url file:///app' | grep "next release"
```

Expected output: `The next release version is 3.1.0` (there were `feat:` commits after `v3.0.0`).

**The Docker Hub secrets.** In Docker Hub: avatar → **Account settings** → **Personal access
tokens** → **Generate new token**, with **Read & Write** access. Copy it right away, because it
is shown only once. Then in GitHub, open the frontend repo → **Settings** → **Secrets and
variables** → **Actions** and create:

- `DOCKERHUB_USERNAME`: your Docker Hub user;
- `DOCKERHUB_TOKEN`: the token you just generated.

Use one token per repo, so you can revoke one without breaking the other pipelines.

**First run.** Push the workflow. If the secrets are not there yet, the `release` job stops at
`Docker Hub authentication`. Thanks to the step order, nothing was committed or tagged, so the
cluster is untouched. Add the secrets and use **Re-run failed jobs**.

**How to check:**

```bash
# git now points to the new version
git pull && grep newTag k8s/kustomization.yaml     # newTag: "3.1.0"
# ArgoCD synced it (or force a refresh, see below)
kubectl -n cicd-system get application front-fintracker \
  -o jsonpath='{.status.sync.status}/{.status.health.status}{"\n"}'
kubectl -n fintracker-dev get deploy front-fintracker-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Expected output: `Synced/Healthy` and `<dockerhub-user>/front-fintracker:3.1.0`.

**ArgoCD polls git about every 3 minutes.** To deploy right away, click **Refresh** on the
Application in the UI, or:

```bash
kubectl -n cicd-system annotate application front-fintracker \
  argocd.argoproj.io/refresh=normal --overwrite
```

The refresh only tells ArgoCD to read git again. Automated sync then applies the change.

### 6. Keep only one old ReplicaSet

Each rollout leaves the previous ReplicaSet behind (with 0 pods) so you can roll back. By
default, a Deployment keeps 10 of them, and the frontend had 6 old ones. To keep only the
current version and the previous one, add this to `k8s/deployment.yaml`:

```yaml
spec:
  replicas: 1
  # Old ReplicaSets kept for rollback (current + 1 previous = 2 versions).
  revisionHistoryLimit: 1
```

The limit counts only the **old** ReplicaSets, so `1` means two in total. Commit it as `chore(k8s): ...`:
`chore` does not trigger a release, and ArgoCD applies the manifest change directly. The field
is not part of the pod template, so the pods do not restart.

**How to check:**

```bash
kubectl -n fintracker-dev get rs -l app=front-fintracker
```

Expected output: two ReplicaSets, the current one (`DESIRED 1`) and the previous one
(`DESIRED 0`).

The concept is explained in detail in
[How to limit the old ReplicaSets kept by a Deployment](how-to-limit-the-old-replicasets-kept-by-a-deployment.md).

## Technical decisions

### A separate Application for the frontend

- **Decision:** a new Application `front-fintracker` pointing to the frontend repo, instead of
  adding the frontend to the API's Application.
- **Why:** the frontend and the API live in different repos and are released separately. With
  one Application each, each one syncs, fails and rolls back on its own, and the ArgoCD UI shows
  the state of each one separately.
- **Alternatives considered:** one Application with multiple sources (`spec.sources`) or an
  "app of apps". Both group things that change at different times, and they add complexity a
  two-app setup does not need.
- **Trade-offs / risks:** two Applications deploy into one namespace, so they must never declare
  the same resource (next decision).

### The Namespace belongs to one Application only

- **Decision:** only the API's manifests declare `Namespace/fintracker-dev`. The frontend's
  kustomization leaves it out.
- **Why:** if two Applications manage the same resource, each one sees the other's changes as
  drift and they keep overwriting each other. ArgoCD also shows a "shared resource" warning.
- **Alternatives considered:** a third Application only for the namespace (and its quota). That is
  cleaner on a bigger setup, but too much for one namespace.
- **Trade-offs / risks:** deleting the API's Application with cascade would delete the Namespace,
  and the frontend with it.

### Pinned tag in git instead of `latest`

- **Decision:** immutable version tags (`3.1.0`) written to `newTag`; `latest` is still pushed,
  but only for convenience outside the cluster.
- **Why:** ArgoCD only deploys what changes in git. A pinned tag also makes every rollback a git
  revert, and `kubectl get deploy` tells you exactly what is running.
- **Alternatives considered:** ArgoCD Image Updater, which watches the registry and writes tags
  back. It is one more component to run and secure, and CI already knows the version.
- **Trade-offs / risks:** every release adds a commit to `main`. Pull before pushing.

### Image first, tag bump second

- **Decision:** the dry run computes the version, the image is pushed, and only then is
  `newTag` committed.
- **Why:** ArgoCD reacts to the git commit. Committing first leaves a window where the cluster
  points to an image that does not exist yet (`ImagePullBackOff`).
- **Alternatives considered:** the API's pipeline commits first and builds after, which is
  simpler but has that window.
- **Trade-offs / risks:** if the real release fails after the push, the image `X.Y.Z` exists in
  Docker Hub with no git tag. The next run computes the same version and pushes over it, so
  nothing breaks.

### GitHub-hosted runner instead of the cluster's self-hosted runner

- **Decision:** `runs-on: ubuntu-latest`.
- **Why:** the self-hosted runner in the cluster is registered only for the API repo (personal
  accounts register runners per repo). For a public repo, GitHub-hosted minutes are free, and
  this pipeline never talks to the cluster: ArgoCD pulls from git.
- **Alternatives considered:** a second runner Deployment registered for the frontend repo. That
  uses cluster CPU and memory for builds, and adds a PAT to look after.
- **Trade-offs / risks:** builds depend on GitHub's runners being available. To deploy, GitHub
  never needs to reach the homelab.

### JavaScript semantic-release instead of python-semantic-release

- **Decision:** `semantic-release` (npm), installed on the fly with `npx -p` at pinned versions.
- **Why:** it is the native tool for a Node project, and it does not need a `pyproject.toml` in
  a JavaScript repo. Both tools follow the same Conventional Commits rules, so the commit
  messages work the same in both repos.
- **Alternatives considered:** adding the plugins to `devDependencies`. That pins them in the
  lockfile, but then every developer installs release tooling they never run.
- **Trade-offs / risks:** the versions are pinned in the workflow, so update them there.

### Polling instead of a GitHub webhook

- **Decision:** keep ArgoCD's default git polling (about 3 minutes), plus a manual refresh when
  needed.
- **Why:** a webhook needs GitHub to reach ArgoCD over the internet. Exposing ArgoCD publicly
  is a big risk to save 3 minutes.
- **Trade-offs / risks:** a deploy shows up a few minutes after the release, not instantly.

## Common problems

| Symptom | Cause | Fix |
|---|---|---|
| First sync replaced the running version | The manifest had `latest` or an old tag (drift) | Before creating the Application, set `newTag` to the image read from the cluster and check with `kubectl diff` |
| `release` job fails at `Docker Hub authentication` | `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` missing in the repo | Add the secrets and use **Re-run failed jobs**; nothing was released |
| semantic-release proposes `1.0.0` | No `vX.Y.Z` tag in the repo | Create a base tag at the deployed version (step 5) |
| No release after a push | Only `chore:`/`ci:`/`docs:` commits since the last tag | Expected; use `fix:` or `feat:` for changes that should ship |
| `newTag` changed in git, but the cluster still runs the old version | ArgoCD has not polled git yet | Wait about 3 minutes or force a refresh (step 5) |
| `kubectl set image` / `rollout undo` is reverted a few seconds later | Self-heal puts git's version back | Change git: revert the release commit or set `newTag` back |
| "Shared resource" warning in ArgoCD | Both Applications declare the same object (e.g. the Namespace) | Keep each resource in one Application only |

## How to undo

- **Roll back a release:** set `newTag` back to the previous version (or `git revert` the
  `chore(release)` commit) and push. ArgoCD deploys the older image, which is still in Docker
  Hub.
- **Stop ArgoCD from managing the frontend, without deleting the workloads:**

  ```bash
  kubectl -n cicd-system patch application front-fintracker --type merge \
    -p '{"metadata":{"finalizers":null}}'
  kubectl -n cicd-system delete application front-fintracker
  ```

  An Application with no `resources-finalizer` is deleted without touching what it deployed.
  Check `metadata.finalizers` before deleting: with the finalizer, the Deployment, Service and
  ConfigMap are deleted too.
- **Remove the pipeline:** delete `.github/workflows/ci.yaml` and `.releaserc.json`, and revoke
  the Docker Hub token.

## References

- [ArgoCD: Declarative setup (Application)](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [ArgoCD: Automated sync policy and self-heal](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Kustomize: images](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/images/)
- [semantic-release](https://semantic-release.gitbook.io/semantic-release/) and
  [@semantic-release/exec](https://github.com/semantic-release/exec)
- [Docker Hub: personal access tokens](https://docs.docker.com/security/for-developers/access-tokens/)
- [Kubernetes: Deployment revision history](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#revision-history-limit)
- Related tutorials: [How to deploy ArgoCD on the Kubernetes cluster](how-to-deploy-argocd-on-the-kubernetes-cluster.md),
  [How to limit the old ReplicaSets kept by a Deployment](how-to-limit-the-old-replicasets-kept-by-a-deployment.md)
