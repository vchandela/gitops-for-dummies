# GitOps for Dummies

A hands-on GitOps setup using **ArgoCD**, **Kargo**, **Helm**, and **kind** — deploying a 4-service microservices app ([Craftista](https://github.com/nitheesh86/microservice-helmcharts)) across dev/staging/prod with automated promotion.

> Based on the [FreeCodeCamp GitOps article](https://www.freecodecamp.org/news/gitops-with-argocd-and-kargo/) — with corrections, kind instead of minikube, and ArgoCD v3 fixes.

---

## What You'll Learn

- GitOps fundamentals: Git as the single source of truth
- **ArgoCD**: continuous sync from Git → Kubernetes, self-heal on drift
- **Kargo**: promotion orchestration — moves a verified image tag through dev → staging → prod by committing to Git
- **Helm**: multi-source Applications splitting chart templates from environment values
- Real gotchas: ArgoCD v3 path restrictions, Kargo RBAC gaps, kind networking quirks

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Source Repos (app code)                                        │
│  microservice-frontend / catalogue / recommendation / voting    │
│         │ push code → GitHub Actions builds image               │
│         ▼                                                        │
│  DockerHub: nitheesh86/microservice-frontend:1.0.11             │
└────────────────────────────┬────────────────────────────────────┘
                             │ Kargo Warehouse polls DockerHub
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Kargo (promotion engine)                                        │
│                                                                  │
│  Warehouse ──► Freight(1.0.11) ──► dev-stage                    │
│                                          │ auto-promote          │
│                                          ▼                       │
│                                    staging-stage                 │
│                                          │ auto/manual           │
│                                          ▼                       │
│                                      prod-stage                  │
│                                                                  │
│  Each promotion: git-clone → yaml-update → git-commit →         │
│  git-push → argocd-update                                       │
└────────────────────────────┬────────────────────────────────────┘
                             │ commits image.tag to env/*/values.yaml
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  This repo (gitops-for-dummies) — the GitOps config repo        │
│                                                                  │
│  env/dev/frontend/frontend-values.yaml   ← Kargo writes here    │
│  env/staging/frontend/frontend-values.yaml                       │
│  env/prod/frontend/frontend-values.yaml                          │
└────────────────────────────┬────────────────────────────────────┘
                             │ ArgoCD watches git
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  ArgoCD (continuous sync)                                        │
│                                                                  │
│  frontend-dev    → namespace: dev   (NodePort 30080)            │
│  frontend-staging → namespace: staging                           │
│  frontend-prod   → namespace: prod                               │
│  ... same for catalogue, recommendation, voting (12 apps total)  │
└─────────────────────────────────────────────────────────────────┘
```

**Key principle:** Kargo never deploys directly. It only commits to Git. ArgoCD detects the commit and deploys. Git is always the source of truth.

---

## Repo Structure

```
gitops-for-dummies/
├── service-charts/              # Helm chart templates (shared across envs)
│   ├── frontend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml          # defaults (overridden per env)
│   │   └── templates/
│   ├── catalogue/
│   ├── recommendation/
│   └── voting/
│
├── env/                         # Per-environment Helm value overrides
│   ├── dev/
│   │   └── frontend/
│   │       └── frontend-values.yaml   ← Kargo writes image.tag here
│   ├── staging/
│   └── prod/
│
├── argocd/
│   └── application/
│       ├── craftista-project.yaml     # ArgoCD AppProject (RBAC boundary)
│       ├── dev/
│       │   ├── frontend.yaml          # ArgoCD Application (multi-source)
│       │   └── ...
│       ├── staging/
│       └── prod/
│
├── kargo/
│   ├── project.yaml                   # Kargo Project (= namespace craftista)
│   ├── projectconfig.yaml             # autopromote label selector
│   ├── frontend-config/
│   │   ├── frontend-warehouse.yaml    # polls DockerHub for new tags
│   │   ├── frontend-stages.yaml       # dev/staging/prod stages + gates
│   │   └── frontend-promotion-tasks.yaml  # 5-step promotion workflow
│   └── ... (catalogue, recommendation, voting)
│
└── kind-cluster.yaml                  # kind cluster config (port mappings)
```

---

## Prerequisites

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) — Kubernetes in Docker
- [OrbStack](https://orbstack.dev/) — Docker runtime (**not Docker Desktop** — Docker Desktop has a networking bug on Mac where kind nodes can't reach `ghcr.io`/`quay.io`. OrbStack fixes it.)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)

---

## Setup

### 1. Create the kind cluster

```bash
kind create cluster --config kind-cluster.yaml
```

Creates a single-node cluster `gitops-for-dummies` with port 30080 mapped for the frontend.

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080  (user: admin, pass: above)
```

### 3. Apply ArgoCD resources

```bash
kubectl apply -f argocd/application/craftista-project.yaml
kubectl apply -k argocd/application/dev/
kubectl apply -k argocd/application/staging/
kubectl apply -k argocd/application/prod/
```

All 12 apps (4 services × 3 envs) should reach Synced + Healthy within a few minutes.

### 4. Install cert-manager (required by Kargo)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=available deployment/cert-manager -n cert-manager --timeout=120s
```

### 5. Install Kargo

```bash
# Generate a bcrypt hash for the admin password
HASH=$(python3 -c "import bcrypt; print(bcrypt.hashpw(b'admin', bcrypt.gensalt(10)).decode())")

helm install kargo \
  oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo \
  --create-namespace \
  --version 1.5.0 \
  --set api.adminAccount.enabled=true \
  --set "api.adminAccount.passwordHash=${HASH}" \
  --set "api.adminAccount.tokenSigningKey=$(openssl rand -hex 20)"

kubectl wait --for=condition=available deployment/kargo-api -n kargo --timeout=120s

# Port-forward UI
kubectl port-forward svc/kargo-api -n kargo 8081:443
# Open: https://localhost:8081  (user: admin, pass: admin)
```

### 6. Fix missing RBAC (Kargo v1.5 bug)

Kargo ships a `ClusterRole` named `kargo-controller-read-secrets` but no binding for it. Without this, Kargo can't read credential secrets and git-push silently fails with "could not read Username."

```bash
kubectl create clusterrolebinding kargo-controller-read-secrets \
  --clusterrole=kargo-controller-read-secrets \
  --serviceaccount=kargo:kargo-controller
```

### 7. Apply Kargo resources

```bash
# Credential secret for git-push
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: craftista
  labels:
    kargo.akuity.io/cred-type: git
type: Opaque
stringData:
  repoURL: https://github.com/YOUR_USERNAME/gitops-for-dummies
  username: YOUR_GITHUB_USERNAME
  password: YOUR_GITHUB_PAT   # needs repo write access
EOF

# All Kargo resources (project, warehouses, stages, promotion tasks)
kubectl apply -k kargo/
```

The Kargo UI should show the `craftista` project with 4 warehouses and 12 stages. With `autopromote: true` set on all stages, any new freight will cascade dev → staging → prod automatically.

---

## How It Works

### ArgoCD Applications — Multi-Source Pattern

ArgoCD v3+ restricts `valueFiles` paths to within the chart directory. Using `../../env/...` (relative paths) doesn't work. Fix: **multi-source Applications** with a `$values` reference:

```yaml
# argocd/application/dev/frontend.yaml
spec:
  sources:
    - repoURL: https://github.com/vchandela/gitops-for-dummies
      path: service-charts/frontend        # the Helm chart
      helm:
        valueFiles:
          - $values/env/dev/frontend/frontend-values.yaml
    - repoURL: https://github.com/vchandela/gitops-for-dummies
      ref: values                          # $values alias
```

### Kargo Promotion Pipeline

Each service has a `PromotionTask` with 5 steps:

```
git-clone → yaml-update(image.tag) → git-commit → git-push → argocd-update
```

`yaml-update` writes exactly one line: `image.tag: <new-version>` into the env's values file. ArgoCD detects the commit and syncs. The promotion task uses the credential secret (step 7) to authenticate the push.

### Freight Verification Gate

Staging won't promote until dev is verified healthy. This is the built-in safety gate:

```
Warehouse creates Freight(1.0.12)
    │
    ▼
dev-stage promotes → ArgoCD deploys → pods healthy?
    │ YES                                  │ NO
    ▼                                      ▼
Freight verified in dev          staging/prod gates blocked
    │
    ▼
staging-stage promotes → healthy?
    │
    ▼
prod-stage promotes
```

### ArgoCD Self-Heal

Manually edit a live resource — e.g., `kubectl scale deployment frontend-dev --replicas=5`. ArgoCD detects the drift from Git and reverts it within ~3 minutes. Git is authoritative; the cluster is always reconciled to match.

---

## Gotchas & Fixes

| # | Symptom | Root Cause | Fix |
|---|---------|-----------|-----|
| 1 | `env/` directory missing from git | Python `.gitignore` template includes `env/` (treats it as virtualenv) | Remove `env/` from `.gitignore`, add `!env/`, then `git add -f env/` |
| 2 | ArgoCD: `no such file or directory` for valueFiles | ArgoCD v3 restricts paths to chart directory; relative `../../env/...` doesn't work | Use multi-source Application with `$values` reference |
| 3 | Kargo git-push: `could not read Username` | `kargo-controller-read-secrets` ClusterRole has no binding in v1.5 | `kubectl create clusterrolebinding kargo-controller-read-secrets ...` |
| 4 | kind nodes can't pull from `ghcr.io` / `quay.io` | Docker Desktop networking bug on Mac | Switch to OrbStack |
| 5 | ArgoCD frontend Service OutOfSync (NodePort) | Kubernetes auto-assigns NodePort; ArgoCD sees drift | Set explicit `nodePort: 30080` in values.yaml |
| 6 | catalogue pods `ImagePullBackOff` | Tag `1.0.2` doesn't exist on DockerHub (article error) | Use `1.0.1` in all `catalogue-values.yaml` files |

---

## Accessing the App

```bash
# Frontend (after cluster setup)
open http://localhost:30080

# ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
open https://localhost:8080

# Kargo UI
kubectl port-forward svc/kargo-api -n kargo 8081:443
open https://localhost:8081
```

---

## References

- [FreeCodeCamp GitOps article](https://www.freecodecamp.org/news/gitops-with-argocd-and-kargo/) (source for this tutorial — contains errors documented above)
- [ArgoCD multi-source Applications](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
- [Kargo docs](https://docs.kargo.io/)
- [kind docs](https://kind.sigs.k8s.io/)
- [OrbStack](https://orbstack.dev/)
