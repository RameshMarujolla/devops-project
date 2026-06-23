# DevOps Project — GitOps K8s Deployment

This repository contains a sample GitOps-style Kubernetes deployment using **kind** (local K8s cluster), **Argo CD** (continuous delivery), and **Kustomize** (environment-specific configuration).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Creating the Cluster](#creating-the-cluster)
4. [Installing Argo CD](#installing-argo-cd)
   - [Accessing the Argo CD UI](#accessing-the-argo-cd-ui)
   - [Retrieving the Admin Password](#retrieving-the-admin-password)
5. [Deploying the Sample Application](#deploying-the-sample-application)
   - [Dev Environment](#dev-environment)
   - [Stage Environment](#stage-environment)
   - [Prod Environment](#prod-environment)
6. [Cleaning Up](#cleaning-up)

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

## Creating the Cluster

Create a local Kubernetes cluster using **kind**:

```bash
kind create cluster --name devops-cluster
```

Verify the cluster is ready:

```bash
kubectl cluster-info
kubectl get nodes
```

---

## Installing Argo CD

### 1. Add the Argo Helm repository

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
```

### 2. Create the `argocd` namespace

```bash
kubectl create namespace argocd
```

### 3. Install Argo CD

```bash
helm install argocd argo/argo-cd --namespace argocd --wait --timeout 10m
```

### 4. Create the ArgoCD Project

The `crossplane-dev` application uses ArgoCD project `platform`. Create it before bootstrapping:

```bash
kubectl apply -f argocd/project-platform.yaml
```

### 5. Verify the installation

```bash
kubectl get pods -n argocd
```

Expected output: all 7 pods should be in `Running` status.

---

## Accessing the Argo CD UI

### Port-forward the Argo CD server

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Then open your browser at: `https://localhost:8080`

> **Note**: Accept the self-signed certificate warning in your browser.

### Retrieving the Admin Password

The default username is `admin`. Retrieve the auto-generated password with:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

> **Security tip**: Delete the initial secret after first login and configure a new password or SSO.

---

## Deploying the Sample Application

The sample application (`myapp`) is deployed using **Kustomize** overlays for each environment.

### Dev Environment

```bash
kubectl apply -k applications/crossplane/overlays/dev/
```

- **Namespace**: `dev`
- **Replicas**: `1`
- **Image tag**: `dev-latest`

### Stage Environment

```bash
kubectl apply -k applications/crossplane/overlays/stage/
```

- **Namespace**: `stage`
- **Replicas**: `1`
- **Image tag**: `stage-latest`

### Prod Environment

```bash
kubectl apply -k applications/crossplane/overlays/prod/
```

- **Namespace**: `prod`
- **Replicas**: `3`
- **Image tag**: `prod-latest`
- **Extras**: HorizontalPodAutoscaler (HPA) included

### Verify Deployments

```bash
kubectl get pods -n dev
kubectl get pods -n stage
kubectl get pods -n prod
```

### Preview Kustomize Output (Dry Run)

Before applying, you can preview the rendered manifests:

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

## Additional Resources

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Helm Documentation](https://helm.sh/docs/)
