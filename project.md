# Project Documentation — DevOps GitOps K8s Deployment

## Overview

This repository implements a **GitOps-style Kubernetes deployment** using:

- **kind** — local Kubernetes cluster
- **Argo CD** — continuous delivery with declarative application definitions
- **Kustomize** — environment-specific configuration overlays
- **Crossplane** — Kubernetes control plane for provisioning cloud infrastructure

The project follows the **App of Apps** pattern with Argo CD, where a single "root" application manages all other applications.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Directory Structure](#directory-structure)
3. [File-by-File Breakdown](#file-by-file-breakdown)
   - [Bootstrap Layer](#bootstrap-layer)
   - [ArgoCD Application Layer](#argocd-application-layer)
   - [Application Base](#application-base)
   - [Environment Overlays](#environment-overlays)
4. [Deployment Flow](#deployment-flow)
5. [Kustomize Layering Model](#kustomize-layering-model)
6. [ArgoCD App of Apps Pattern](#argocd-app-of-apps-pattern)
7. [Environment Configuration Matrix](#environment-configuration-matrix)
8. [GitOps Workflow](#gitops-workflow)
9. [Operations](#operations)
10. [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        GitHub Repo                          │
│  ┌──────────────┐  ┌──────────────────────────────────┐     │
│  │ bootstrap/   │  │ applications/crossplane/         │     │
│  │ root-app.yaml│  │ ├── base/                        │     │
│  └──────┬───────┘  │ │   ├── kustomization.yaml       │     │
│         │          │ │   ├── namespace.yaml           │     │
│         │          │ │   └── values.yaml              │     │
│         │          │ └── overlays/                    │     │
│         │          │     ├── dev/                     │     │
│         │          │     ├── stage/                   │     │
│         │          │     └── prod/                    │     │
│         │          └──────────────────────────────────┘     │
│         │                                                   │
│  ┌──────┴─────────────────────────────────────────────┐    │
│  │ argocd/apps-dev/                                   │    │
│  │ └── crossplane.yaml  ← ArgoCD Application manifest │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────┘
                             │ watches (Git polling/webhook)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    Argo CD (in-cluster)                     │
│                                                             │
│  ┌─────────────────┐                                       │
│  │  root-app       │  ← Application-of-Applications        │
│  │  (bootstrap)    │                                       │
│  └────────┬────────┘                                       │
│           │ points to argocd/apps-dev/                      │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │  crossplane-dev │  ← Application                        │
│  │  (argocd layer) │                                       │
│  └────────┬────────┘                                       │
│           │ points to applications/crossplane/overlays/dev  │
│           ▼                                                 │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Kustomize renders final manifests           │   │
│  │   (base + dev overlay = complete dev deployment)    │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │ kubectl apply -k
                             ▼
                    ┌─────────────────┐
                    │   Kubernetes    │
                    │  (kind cluster) │
                    └─────────────────┘
```

---

## Directory Structure

```
.
├── README.md                               # Project documentation & setup guide
├── project.md                              # This file — complete architecture reference
│
├── bootstrap/
│   └── dev/
│       └── root-app.yaml                   # ArgoCD root application (App of Apps)
│
├── argocd/
│   ├── apps-dev/
│   │   └── crossplane.yaml                 # ArgoCD Application: crossplane-dev
│   └── project-platform.yaml               # ArgoCD AppProject: platform
│
└── applications/
    └── crossplane/
        ├── base/                           # Shared base configuration
        │   ├── kustomization.yaml          # Base Kustomize config (helm chart + namespace)
        │   ├── namespace.yaml              # crossplane-system namespace definition
        │   └── values.yaml                 # Base Helm values for Crossplane
        │
        └── overlays/                       # Environment-specific patches
            ├── dev/
            │   ├── kustomization.yaml      # Dev Kustomize config
            │   └── values-dev.yaml         # Dev-specific Helm values overrides
            ├── stage/
            │   ├── deployment-patch.yaml   # Stage Deployment resource patch
            │   └── kustomization.yaml      # Stage Kustomize config
            └── prod/
                ├── deployment-patch.yaml   # Prod Deployment resource patch
                ├── hpa.yaml                # Prod HorizontalPodAutoscaler
                └── kustomization.yaml      # Prod Kustomize config
```

---

## File-by-File Breakdown

### Bootstrap Layer

#### `bootstrap/dev/root-app.yaml`

**Purpose:** The top-level ArgoCD "root" application that implements the **App of Apps** pattern. This is the only manifest you manually apply with `kubectl`. Once running, ArgoCD will automatically discover and manage all other applications defined in the `argocd/apps-dev/` directory.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.name` | `root-app` | Name of the root application |
| `spec.project` | `default` | ArgoCD project scope |
| `spec.source.repoURL` | GitHub repo URL | Source of truth |
| `spec.source.targetRevision` | `dev` (or `main`) | Git branch to track |
| `spec.source.path` | `argocd/apps-dev` | Directory containing child Application manifests |
| `spec.destination` | `https://kubernetes.default.svc` | In-cluster API server |
| `spec.syncPolicy.automated.prune` | `true` | Delete resources removed from Git |
| `spec.syncPolicy.automated.selfHeal` | `true` | Revert manual changes to match Git |
| `spec.syncPolicy.syncOptions` | `CreateNamespace=true` | Auto-create target namespace |

**Deploy command:**
```bash
kubectl apply -f bootstrap/dev/root-app.yaml
```

---

### ArgoCD Application Layer

#### `argocd/apps-dev/crossplane.yaml`

**Purpose:** Defines the Crossplane application managed by ArgoCD. This is a child application referenced by the root-app.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.name` | `crossplane-dev` | Application name |
| `metadata.namespace` | `argocd` | ArgoCD's own namespace |
| `spec.project` | `platform` | ArgoCD project scope |
| `spec.source.repoURL` | GitHub repo URL | Source of truth |
| `spec.source.targetRevision` | `dev` | Git branch to track |
| `spec.source.path` | `applications/crossplane/overlays/dev` | Kustomize overlay path |
| `spec.destination.server` | `https://kubernetes.default.svc` | In-cluster API server |
| `spec.destination.namespace` | `crossplane-system` | Target namespace for Crossplane |
| `spec.syncPolicy.automated.prune` | `true` | Delete resources removed from Git |
| `spec.syncPolicy.automated.selfHeal` | `true` | Revert manual changes to match Git |

---

#### `argocd/project-platform.yaml`

**Purpose:** Creates the ArgoCD `platform` AppProject that scopes the `crossplane-dev` application. By default ArgoCD only has the `default` project. This manifest must be applied **before** bootstrapping the root-app, or ArgoCD will refuse to sync the `crossplane-dev` application.

| Field | Value | Description |
|-------|-------|-------------|
| `metadata.name` | `platform` | Project name referenced by `crossplane-dev` |
| `metadata.namespace` | `argocd` | ArgoCD's own namespace |
| `spec.description` | `Platform infrastructure components...` | Human-readable description |
| `spec.destinations` | `namespace: '*', server: 'https://kubernetes.default.svc'` | Allows deploying to any namespace on the in-cluster API |
| `spec.sourceRepos` | `'*'` | Allows any Git repository as a source |

**Why it's needed:** The `crossplane-dev` Application specifies `spec.project: platform`. ArgoCD's RBAC model requires that project to exist before any Application can use it. Without this manifest, the ArgoCD UI shows:

> *"Unable to load data: app is not allowed in project 'platform', or the project does not exist"*

**Deploy command:**
```bash
kubectl apply -f argocd/project-platform.yaml
```

---

### Application Base

#### `applications/crossplane/base/kustomization.yaml`

**Purpose:** Defines the shared base configuration for Crossplane deployment using Kustomize's `helmCharts` plugin.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: crossplane-system

resources:
  - namespace.yaml

helmCharts:
  - name: crossplane
    repo: https://charts.crossplane.io/stable
    version: 1.20.1              # Only change this on upgrade
    releaseName: crossplane
    namespace: crossplane-system
    valuesFile: values.yaml

generatorOptions:
  disableNameSuffixHash: true
```

**Key settings:**
- **Namespace:** All resources go into `crossplane-system`
- **Helm chart:** Crossplane v1.20.1 from official stable repo
- **Generator option:** Disables hash suffix on generated ConfigMaps/Secrets for stable naming

---

#### `applications/crossplane/base/namespace.yaml`

**Purpose:** Creates the `crossplane-system` namespace if it doesn't exist.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: crossplane-system
```

---

#### `applications/crossplane/base/values.yaml`

**Purpose:** Default Helm values for Crossplane deployment. These are the base values used across all environments unless overridden.

| Setting | Base Value | Description |
|---------|-----------|-------------|
| `provider.packages` | `[]` | No providers installed by default |
| `resourcesCrossplane.limits.cpu` | `500m` | Crossplane pod CPU limit |
| `resourcesCrossplane.limits.memory` | `512Mi` | Crossplane pod memory limit |
| `resourcesCrossplane.requests.cpu` | `100m` | Crossplane pod CPU request |
| `resourcesCrossplane.requests.memory` | `256Mi` | Crossplane pod memory request |
| `resourcesRBACManager.limits.cpu` | `100m` | RBAC manager CPU limit |
| `resourcesRBACManager.limits.memory` | `512Mi` | RBAC manager memory limit |
| `resourcesRBACManager.requests.cpu` | `100m` | RBAC manager CPU request |
| `resourcesRBACManager.requests.memory` | `256Mi` | RBAC manager memory request |
| `metrics.enabled` | `true` | Prometheus metrics enabled |
| `leaderElection.enabled` | `true` | HA leader election enabled |

---

### Environment Overlays

#### Dev Overlay: `applications/crossplane/overlays/dev/`

**Purpose:** Lightweight development configuration with reduced resources and disabled features.

**`kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

helmCharts:
  - name: crossplane
    repo: https://charts.crossplane.io/stable
    version: 1.20.1
    releaseName: crossplane
    namespace: crossplane-system
    valuesFile: ../../base/values.yaml      # Base defaults
    additionalValuesFiles:
      - values-dev.yaml                     # Dev overrides on top
```

**`values-dev.yaml` overrides:**

| Setting | Base Value | Dev Override | Effect |
|---------|-----------|-------------|--------|
| `resourcesCrossplane.limits.cpu` | `500m` | `250m` | Reduced CPU limit |
| `resourcesCrossplane.limits.memory` | `512Mi` | `256Mi` | Reduced memory limit |
| `resourcesCrossplane.requests.memory` | `256Mi` | `128Mi` | Reduced memory request |
| `leaderElection.enabled` | `true` | `false` | Disabled (single replica) |
| `metrics.enabled` | `true` | `false` | Disabled for dev |

**Dev characteristics:**
- Single replica, no HA
- Minimal resource footprint
- No metrics scraping
- Fast startup, low overhead

---

#### Stage Overlay: `applications/crossplane/overlays/stage/`

**Purpose:** Staging environment with moderate configuration. Includes sample Deployment patches for reference (though primarily Helm-based in this project).

**`kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: stage

namePrefix: stage-

resources:
  - ../../base

replicas:
  - name: app
    count: 2

images:
  - name: myapp
    newTag: stage-latest

patches:
  - path: deployment-patch.yaml
```

**`deployment-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

**Stage characteristics:**
- Namespace: `stage`
- Name prefix: `stage-` (resources named `stage-app`)
- Replicas: 2
- Image tag: `stage-latest`
- Resource limits: 200m CPU / 256Mi memory

---

#### Prod Overlay: `applications/crossplane/overlays/prod/`

**Purpose:** Production environment with high availability, horizontal autoscaling, and strict resource limits.

**`kustomization.yaml`:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod

namePrefix: prod-

resources:
  - ../../base

replicas:
  - name: app
    count: 3

images:
  - name: myapp
    newTag: prod-latest

patches:
  - path: deployment-patch.yaml
  - path: hpa.yaml
```

**`deployment-patch.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

**`hpa.yaml` (HorizontalPodAutoscaler):**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Prod characteristics:**
- Namespace: `prod`
- Name prefix: `prod-`
- Replicas: 3 (base)
- Image tag: `prod-latest`
- Resource limits: 500m CPU / 512Mi memory
- **HPA:** Auto-scales from 3 to 10 replicas based on 70% CPU utilization

---

## Deployment Flow

The deployment follows a **declarative, layered approach**:

### Step 1: Cluster Setup
```bash
kind create cluster --name devops-cluster
```

### Step 2: ArgoCD Installation
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd --wait --timeout 10m
```

### Step 2B: Create the ArgoCD `platform` Project

The `crossplane-dev` application is scoped to ArgoCD project `platform` (see `spec.project` in `argocd/apps-dev/crossplane.yaml`). You must create this project before bootstrapping, or ArgoCD will refuse to sync with the error:

> *"app is not allowed in project 'platform', or the project does not exist"*

```bash
kubectl apply -f argocd/project-platform.yaml
```

**`argocd/project-platform.yaml`:**
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

### Step 3: Bootstrap Root Application
```bash
kubectl apply -f bootstrap/dev/root-app.yaml
```

**What happens:**
1. ArgoCD creates the `root-app` Application
2. `root-app` scans the `argocd/apps-dev/` directory
3. Finds `crossplane.yaml` and creates the `crossplane-dev` Application
4. `crossplane-dev` points to `applications/crossplane/overlays/dev/`
5. Kustomize renders the final manifests (base + dev overlay)
6. ArgoCD applies the rendered manifests to `crossplane-system` namespace
7. Crossplane pods start running

### Step 4: Verify
```bash
kubectl get pods -n crossplane-system
kubectl get applications -n argocd
```

---

## Kustomize Layering Model

```
┌──────────────────────────────────────────────┐
│           OVERLAY (Environment)              │
│  ┌────────────────────────────────────────┐  │
│  │  Environment-specific overrides:       │  │
│  │  • namespace                           │  │
│  │  • namePrefix                          │  │
│  │  • replicas                            │  │
│  │  • images                              │  │
│  │  • patches (deployment, HPA)           │  │
│  │  • additionalValuesFiles (Helm values) │  │
│  └────────────────────┬───────────────────┘  │
└───────────────────────┼──────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────┐
│              BASE (Shared)                   │
│  ┌────────────────────────────────────────┐  │
│  │  Common configuration:                 │  │
│  │  • namespace.yaml                      │  │
│  │  • helmCharts (Crossplane v1.20.1)     │  │
│  │  • values.yaml (default Helm values)   │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**Merge strategy:**
- Overlays **extend** the base by referencing it via `resources: [../../base]`
- Helm values are layered: `base/values.yaml` + overlay's `additionalValuesFiles`
- Kustomize patches modify specific fields in base resources
- `namePrefix` prepends a prefix to all resource names in that overlay

---

## ArgoCD App of Apps Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                      ROOT APPLICATION                         │
│                    (bootstrap/dev/root-app.yaml)              │
│                                                               │
│  source.path: argocd/apps-dev/                                │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────┐  │
│  │ crossplane.yaml │  │  (future apps)  │  │  (future)   │  │
│  │     (app)       │  │                 │  │             │  │
│  └────────┬────────┘  └─────────────────┘  └─────────────┘  │
└───────────┼───────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│                  CHILD APPLICATIONS                           │
│                                                               │
│  crossplane-dev  →  source.path: applications/crossplane/     │
│                         overlays/dev                          │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Kustomize render                        │    │
│  │  (base + dev overlay)                                │    │
│  └──────────────────────┬──────────────────────────────┘    │
└─────────────────────────┼─────────────────────────────────────┘
                          ▼
                  ┌───────────────┐
                  │  Kubernetes   │
                  │   Cluster     │
                  └───────────────┘
```

**Benefits:**
- Single bootstrap command deploys everything
- Adding new apps only requires creating a new file in `argocd/apps-dev/`
- ArgoCD UI shows the entire application hierarchy
- Deleting the root-app cascades deletion to all child apps

---

## Environment Configuration Matrix

| Setting | Dev | Stage | Prod |
|---------|-----|-------|------|
| **Namespace** | `crossplane-system` | `stage` | `prod` |
| **Name Prefix** | (none) | `stage-` | `prod-` |
| **Replicas** | 1 (implied) | 2 | 3 + HPA (3-10) |
| **Image Tag** | (Helm default) | `stage-latest` | `prod-latest` |
| **CPU Limit** | `250m` | `200m` | `500m` |
| **Memory Limit** | `256Mi` | `256Mi` | `512Mi` |
| **CPU Request** | `100m` | `100m` | `250m` |
| **Memory Request** | `128Mi` | `128Mi` | `256Mi` |
| **Leader Election** | `false` | (base: true) | (base: true) |
| **Metrics** | `false` | (base: true) | (base: true) |
| **HPA** | ❌ No | ❌ No | ✅ Yes (70% CPU) |

---

## GitOps Workflow

### Normal Operations (Day-to-Day)

1. **Developer makes changes** → Commit and push to `dev` branch
2. **ArgoCD detects changes** → Polls Git every 3 minutes (or via webhook)
3. **ArgoCD syncs** → Automatically applies changes to the cluster
4. **Drift detection** → If someone manually changes the cluster, ArgoCD self-heals back to Git state

### Promoting to Higher Environments

**Option A: Branch-based promotion**
```bash
# Promote dev → stage
git checkout stage
git merge dev
git push origin stage

# Update ArgoCD to track stage branch
# (Modify targetRevision in application manifests)
```

**Option B: Path-based promotion (same branch)**
```bash
# Copy dev config to stage
# Modify targetRevision to 'main' and path to 'overlays/stage'
```

### Adding a New Environment

1. Create new overlay directory:
   ```bash
   mkdir applications/crossplane/overlays/qa
   cp -r applications/crossplane/overlays/dev/* applications/crossplane/overlays/qa/
   ```

2. Modify `kustomization.yaml` and values for QA environment

3. Create ArgoCD Application manifest:
   ```bash
   cp argocd/apps-dev/crossplane.yaml argocd/apps-dev/crossplane-qa.yaml
   ```
   Update `metadata.name`, `spec.source.path`, and `spec.destination.namespace`

4. Commit and push — ArgoCD will auto-discover the new application within 3 minutes

---

## Operations

### Preview Rendered Manifests (Dry Run)

```bash
# Dev
kubectl kustomize applications/crossplane/overlays/dev/

# Stage
kubectl kustomize applications/crossplane/overlays/stage/

# Prod
kubectl kustomize applications/crossplane/overlays/prod/
```

### Manual Apply (without ArgoCD)

```bash
kubectl apply -k applications/crossplane/overlays/dev/
kubectl apply -k applications/crossplane/overlays/stage/
kubectl apply -k applications/crossplane/overlays/prod/
```

### View ArgoCD Applications

```bash
kubectl get applications -n argocd
kubectl describe application root-app -n argocd
kubectl describe application crossplane-dev -n argocd
```

### Port-Forward ArgoCD UI

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

### Get ArgoCD Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## Troubleshooting

### Check Application Sync Status

```bash
kubectl get application root-app -n argocd -o yaml
kubectl get application crossplane-dev -n argocd -o yaml
```

Look for `status.sync.status` and `status.health.status`:
- `Synced` + `Healthy` = ✅ Good
- `OutOfSync` + `Healthy` = ⚠️ Changes pending sync
- `Synced` + `Degraded` = ❌ Application failing

### Check Kustomize Build Output

```bash
kubectl kustomize applications/crossplane/overlays/dev/ 2>&1 | head -50
```

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| `namespace not found` | Namespace missing | Ensure `namespace.yaml` is in base resources or `CreateNamespace=true` in syncOptions |
| `helm chart not found` | Repo not added | Verify `helm repo add` command in setup |
| `resource conflict` | namePrefix collision | Ensure unique prefixes per environment |
| HPA not scaling | Metrics server missing | Install metrics-server in cluster |
| ArgoCD not syncing | Wrong branch/path | Check `targetRevision` and `path` in Application spec |

### Git Push Failed: Diverged Branches

**Error:** `fatal: refusing to merge unrelated histories`

**Resolution:**
```bash
git fetch origin
git merge origin/main --allow-unrelated-histories
# Resolve conflicts, then:
git add <resolved-files>
git commit -m "Merge origin/main"
git push origin main
```

---

## Technology Versions

| Component | Version |
|-----------|---------|
| Crossplane Helm Chart | 1.20.1 |
| Kustomize API | kustomize.config.k8s.io/v1beta1 |
| ArgoCD API | argoproj.io/v1alpha1 |
| Kubernetes API (Deployment) | apps/v1 |
| Kubernetes API (HPA) | autoscaling/v2 |
| Kubernetes API (Namespace) | v1 |

---

## Additional Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Helm Documentation](https://helm.sh/docs/)
- [Crossplane Documentation](https://docs.crossplane.io/)
