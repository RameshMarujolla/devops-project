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
├── README.md
└── applications/
    └── crossplane/
        ├── base/
        │   ├── deployment.yaml      # Base Deployment manifest
        │   ├── service.yaml         # Base Service manifest
        │   └── kustomization.yaml   # Base Kustomize config
        └── overlays/
            ├── dev/
            │   ├── deployment-patch.yaml
            │   └── kustomization.yaml   # Dev environment overrides
            ├── stage/
            │   ├── deployment-patch.yaml
            │   └── kustomization.yaml   # Stage environment overrides
            └── prod/
                ├── deployment-patch.yaml
                ├── hpa.yaml             # HorizontalPodAutoscaler for prod
                └── kustomization.yaml   # Prod environment overrides
```

- **Base**: Shared Kubernetes manifests used across all environments.
- **Overlays**: Environment-specific patches and configurations (replica count, image tag, namespace, HPA, etc.).

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

## Additional Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Helm Documentation](https://helm.sh/docs/)
