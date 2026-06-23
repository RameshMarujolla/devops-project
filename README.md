# DevOps Project — GitOps K8s Deployment

This repository contains a sample GitOps-style Kubernetes deployment using **kind** (local K8s cluster), **Argo CD** (continuous delivery), and **Kustomize** (environment-specific configuration).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Manual Setup Steps (Run Once)](#manual-setup-steps-run-once)
   - [Step 1 — Create the Cluster](#step-1--create-the-cluster)
   - [Step 2 — Install Argo CD](#step-2--install-argo-cd)
   - [Step 3 — Create the ArgoCD `platform` Project](#step-3--create-the-argocd-platform-project)
   - [Step 4 — Bootstrap the Root Application](#step-4--bootstrap-the-root-application)
   - [Step 5 — Verify Everything Synced](#step-5--verify-everything-synced)
4. [What ArgoCD Automates After Bootstrap](#what-argocd-automates-after-bootstrap)
5. [Manual Fallback (Without ArgoCD)](#manual-fallback-without-argocd)
   - [Dev Environment](#dev-environment)
   - [Stage Environment](#stage-environment)
   - [Prod Environment](#prod-environment)
6. [Accessing the Argo CD UI](#accessing-the-argo-cd-ui)
7. [Cleaning Up](#cleaning-up)
8. [Issue Resolution Log](#issue-resolution-log)
9. [Additional Resources](#additional-resources)

---

## Prerequisites

Ensure the following tools are installed on your machine:

| Tool | Purpose | Installation |
|------|---------|--------------|
| `kubectl` | Kubernetes CLI | https://kubernetes.io/docs/tasks/tools/ |
| `helm` | Kubernetes package manager | https://helm.sh/docs/intro/install/ |
| `kind` | Local Kubernetes clusters | https://kind.sigs.k8s.io/docs/user/quick-start/ |

Verify your installations:

```bash
kubectl version --client
helm version
kind version
```

---

## Project Structure

```
.
├── README.md                               # Project documentation & setup guide
├── project.md                              # Complete architecture reference
│
├── bootstrap/
│   └── dev/
│       ├── project-platform.yaml           # ArgoCD AppProject: platform
│       └── root-app.yaml                   # ArgoCD root application (App of Apps)
│
├── argocd/
│   └── apps-dev/
│       └── crossplane.yaml                 # ArgoCD Application: crossplane-dev
│
└── applications/
    └── crossplane/
        ├── base/                           # Shared base configuration
        │   ├── kustomization.yaml          # Base Kustomize config
        │   ├── namespace.yaml              # crossplane-system namespace
        │   └── values.yaml                 # Base Helm values for Crossplane
        │
        └── overlays/                       # Environment-specific patches
            ├── dev/
            │   ├── kustomization.yaml      # Dev Kustomize config
            │   └── values-dev.yaml         # Dev-specific Helm values
            ├── stage/
            │   ├── deployment-patch.yaml   # Stage Deployment resource patch
            │   └── kustomization.yaml      # Stage Kustomize config
            └── prod/
                ├── deployment-patch.yaml   # Prod Deployment resource patch
                ├── hpa.yaml                # HorizontalPodAutoscaler for prod
                └── kustomization.yaml      # Prod Kustomize config
```

- **Base**: Shared Kubernetes manifests (namespace, Helm chart, default values) used across all environments.
- **Overlays**: Environment-specific patches and configurations (resource limits, replica count, image tag, HPA, etc.).
- **argocd/**: ArgoCD Application manifests that define what gets deployed.
- **bootstrap/**: One-time setup manifests (AppProject + root App-of-Apps) that must be applied manually.

---

## Manual Setup Steps (Run Once)

These are the **only commands you run manually**. After Step 4, ArgoCD takes over and manages everything automatically via GitOps.

### Step 1 — Create the Cluster

Create a local Kubernetes cluster using **kind**:

```bash
kind create cluster --name devops-cluster
```

Verify the cluster is ready:

```bash
kubectl cluster-info
kubectl get nodes
```

### Step 2 — Install Argo CD

```bash
# 2a. Add the Argo Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo

# 2b. Create the argocd namespace
kubectl create namespace argocd

# 2c. Install ArgoCD via Helm
helm install argocd argo/argo-cd --namespace argocd --wait --timeout 10m

# 2d. Verify ArgoCD pods are running
kubectl get pods -n argocd
```

Expected output: all ArgoCD pods should be in `Running` status.

### Step 3 — Create the ArgoCD `platform` Project

The `root-app` and `crossplane-dev` applications both use ArgoCD project `platform`. You must create this project before applying the root application.

```bash
kubectl apply -f bootstrap/dev/project-platform.yaml
```

### Step 4 — Bootstrap the Root Application

Apply the single bootstrap manifest. This is the **last manual setup step**.

```bash
kubectl apply -f bootstrap/dev/root-app.yaml
```

### Step 5 — Verify Everything Synced

```bash
kubectl get applications -n argocd
```

You should see both `root-app` and `crossplane-dev` in `Synced` / `Healthy` state.

---

## What ArgoCD Automates After Bootstrap

Once `root-app.yaml` is applied, ArgoCD handles everything automatically. No further manual commands are needed for normal operations.

| Step | Who does it | What happens |
|------|-------------|--------------|
| ① | ArgoCD | `root-app` scans `argocd/apps-dev/` |
| ② | ArgoCD | `root-app` discovers `crossplane.yaml` and creates the `crossplane-dev` Application |
| ③ | ArgoCD | `crossplane-dev` points to `applications/crossplane/overlays/dev/` |
| ④ | Kustomize | Renders final manifests (base + dev overlay) |
| ⑤ | ArgoCD | Applies manifests to `crossplane-system` namespace |
| ⑥ | GitOps | On every Git push to `dev` branch, ArgoCD auto-syncs changes |
| ⑦ | Drift detection | If someone manually changes the cluster, ArgoCD self-heals back to Git state |

---

## Manual Fallback (Without ArgoCD)

If you need to deploy directly without ArgoCD, use `kubectl apply -k`.

### Dev Environment

```bash
kubectl apply -k applications/crossplane/overlays/dev/
```

### Stage Environment

```bash
kubectl apply -k applications/crossplane/overlays/stage/
```

### Prod Environment

```bash
kubectl apply -k applications/crossplane/overlays/prod/
```

### Preview Kustomize Output (Dry Run)

```bash
kubectl kustomize applications/crossplane/overlays/dev/
```

---

## Cleaning Up

### Remove the Application

```bash
kubectl delete -k applications/crossplane/overlays/dev/
kubectl delete -k applications/crossplane/overlays/stage/
kubectl delete -k applications/crossplane/overlays/prod/
```

### Uninstall Argo CD

```bash
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

### Delete the Cluster

```bash
kind delete cluster --name devops-cluster
```

---

## Issue Resolution Log

### Git Push Failed: Diverged Branches with Unrelated Histories

**Error:**
```
fatal: refusing to merge unrelated histories
```

**Context:**
Local `main` and `origin/main` had diverged with **no common ancestor** (unrelated histories). The branches each had one commit not present in the other.

**Resolution Steps:**

1. Fetch remote changes:
   ```bash
   git fetch origin
   ```

2. Merge with `--allow-unrelated-histories`:
   ```bash
   git merge origin/main --allow-unrelated-histories
   ```

3. Resolve merge conflicts. In this case, `README.md` had a conflict (local had content while remote had only a stub). The conflict was resolved by keeping the local `README.md` content.

4. Stage the resolved file:
   ```bash
   git add README.md
   ```

5. Complete the merge commit:
   ```bash
   git commit -m "Merge origin/main"
   ```

6. Push to remote:
   ```bash
   git push origin main
   ```

**Root Cause:** The local repository was initialized independently of the remote GitHub repository, resulting in two unrelated commit histories that could not be merged without the `--allow-unrelated-histories` flag.

---

### ArgoCD Error: App Not Allowed in Project "platform"

**Error (in ArgoCD UI):**
```
Unable to load data: app is not allowed in project "platform", or the project does not exist
```

**Context:**
The `crossplane-dev` ArgoCD Application specifies `spec.project: platform`, but ArgoCD only ships with the built-in `default` project. The `platform` project must be explicitly created before ArgoCD will allow the application to sync.

**Resolution Steps:**

1. Create the ArgoCD `platform` AppProject:
   ```bash
   kubectl apply -f bootstrap/dev/project-platform.yaml
   ```

2. The `bootstrap/dev/project-platform.yaml` file:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AppProject
   metadata:
     name: platform
     namespace: argocd
   spec:
     description: Platform infrastructure components (Crossplane, etc.)
     destinations:
       - namespace: '*'
         server: https://kubernetes.default.svc
     sourceRepos:
       - '*'
   ```

3. Refresh the application in ArgoCD UI (or wait for auto-sync).

**Root Cause:** ArgoCD requires AppProjects to exist before any Application can reference them. The `platform` project was referenced in `argocd/apps-dev/crossplane.yaml` but had not been created in the cluster yet.

---

### ArgoCD Error: "revision dev must be resolved"

**Error (in ArgoCD UI):**
```
Unable to load data: revision dev must be resolved
```

**Context:**
Both `root-app.yaml` and `crossplane.yaml` specified `targetRevision: dev`, but the `dev` branch did not exist on the remote GitHub repository. Only `main` existed.

**Resolution Steps:**

1. Create a local `dev` branch from `main`:
   ```bash
   git checkout -b dev
   ```

2. Push the `dev` branch to remote:
   ```bash
   git push origin dev
   ```

**Root Cause:** ArgoCD polls the Git repository for the branch specified in `targetRevision`. If the branch does not exist, it cannot resolve the revision and the sync fails.

---

### ArgoCD Error: Kustomize Build Failed, "must specify --enable-helm"

**Error (ArgoCD application conditions):**
```
trouble configuring builtin HelmChartInflationGenerator with config: ...
: must specify --enable-helm
```

**Context:**
The dev overlay uses Kustomize's `helmCharts` plugin to render the Crossplane Helm chart. ArgoCD's Kustomize support does **not** enable Helm by default for security reasons. Without `--enable-helm`, Kustomize refuses to process `helmCharts` blocks.

**Resolution Steps:**

1. Patch the ArgoCD ConfigMap to enable Helm for Kustomize:
   ```bash
   kubectl patch configmap argocd-cm -n argocd --type merge \
     -p '{"data":{"kustomize.buildOptions":"--enable-helm"}}'
   ```

2. Restart the ArgoCD repo-server so it picks up the new config:
   ```bash
   kubectl rollout restart deployment argocd-repo-server -n argocd
   ```

**Root Cause:** ArgoCD repo-server runs Kustomize with a restricted set of flags. The `helmCharts` Kustomize plugin requires the `--enable-helm` flag, which must be explicitly enabled via the `argocd-cm` ConfigMap.

---

### Kustomize Error: valuesFile Outside Overlay Directory

**Error (ArgoCD repo-server logs):**
```
security; file '<path>/applications/crossplane/base/values.yaml' is not in or below
'<path>/applications/crossplane/overlays/dev'
```

**Context:**
The dev overlay's `kustomization.yaml` contained a duplicate `helmCharts` block with `valuesFile: ../../base/values.yaml`. Kustomize's `--enable-helm` enforces a security restriction: all referenced files must be within or below the kustomization directory (`overlays/dev/` in this case). The path `../../base/values.yaml` escapes the overlay directory.

Additionally, the base `kustomization.yaml` **already** contained a `helmCharts` block. Having `helmCharts` in both the base and the overlay causes Crossplane to be rendered **twice**, producing duplicate resources.

**Resolution:**

Removed the duplicate `helmCharts` block from `applications/crossplane/overlays/dev/kustomization.yaml`. The base renders the Helm chart once with `base/values.yaml`; all overlays simply reference the base and apply patches.

**Before (broken):**
```yaml
resources:
  - ../../base

helmCharts:
  - name: crossplane
    valuesFile: ../../base/values.yaml      # escapes overlay dir
    additionalValuesFiles:
      - values-dev.yaml
```

**After (fixed):**
```yaml
resources:
  - ../../base
```

**Root Cause:** Kustomize's `--enable-helm` restricts file access to the kustomization directory tree for security. Paths like `../../base/values.yaml` violate this boundary. The correct pattern is to render the Helm chart only in the base, and use Kustomize patches (not Helm values) for overlay-specific changes.

---

### ArgoCD Error: Cluster-Scoped Resources Not Permitted

**Error (ArgoCD application sync result):**
```
resource rbac.authorization.k8s.io:ClusterRole is not permitted in project platform
resource rbac.authorization.k8s.io:ClusterRoleBinding is not permitted in project platform
```

**Context:**
The Crossplane Helm chart creates cluster-scoped resources: `ClusterRole`, `ClusterRoleBinding`, `CustomResourceDefinition`, and `Namespace`. By default, ArgoCD AppProjects only allow namespaced resources (via `destinations`). Cluster-scoped resources require explicit whitelist configuration.

**Resolution:**

Added `clusterResourceWhitelist` to `bootstrap/dev/project-platform.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Platform infrastructure components (Crossplane, etc.)
  destinations:
    - namespace: '*'
      server: https://kubernetes.default.svc
  sourceRepos:
    - '*'
  clusterResourceWhitelist:       # ← added
    - group: '*'
      kind: '*'
```

**Root Cause:** `destinations` with `namespace: '*'` only permits **namespaced** resources in any namespace. Cluster-scoped resources (`ClusterRole`, `ClusterRoleBinding`, `CustomResourceDefinition`, `Namespace`) require `clusterResourceWhitelist` in the AppProject definition.

---

### Crossplane CrashLoopBackOff: leaderElection Map Value

**Error (Crossplane container logs):**
```
crossplane: error: --leader-election: bool value must be true, 1, yes, false, 0 or no
but got "map[enabled:true]"
(from envar LEADER_ELECTION="map[enabled:true]")
```

**Context:**
The `values.yaml` file used the nested-map syntax for `leaderElection`:
```yaml
leaderElection:
  enabled: true
```

Crossplane v1.20.1 expects `leaderElection` to be a flat boolean, not a nested map. The Helm chart template incorrectly rendered the Go map `map[enabled:true]` into the `LEADER_ELECTION` environment variable, causing the Crossplane binary to crash immediately on startup.

**Resolution:**

Changed `applications/crossplane/base/values.yaml` from:
```yaml
leaderElection:
  enabled: true
```

To:
```yaml
leaderElection: true
```

**Root Cause:** Crossplane Helm chart v1.20.1 expects `leaderElection` as a top-level boolean key. The nested `enabled` property is valid in some newer or older chart versions, but v1.20.1's template logic treats the entire map as a string and passes it to the container as a malformed environment variable.

---

## Additional Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Helm Documentation](https://helm.sh/docs/)
