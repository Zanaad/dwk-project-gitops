# dwk-project-gitops

Infrastructure as Code and Kubernetes manifests for the Todo Project, following GitOps principles. This repository contains all Kubernetes configurations, Kustomize overlays, and ArgoCD applications.

**Related Repository:** [dwk-project-code](https://github.com/Zanaad/dwk-project-code) - Application source code and CI/CD workflows

## 📁 Repository Structure

```
dwk-project-gitops/
├── base/                      # Base Kubernetes manifests (shared across environments)
│   ├── broadcaster/k8s/       # Broadcaster service manifests
│   ├── todo-app/k8s/          # Todo App (frontend) manifests
│   ├── todo-backend/k8s/      # Todo Backend API + PostgreSQL manifests
│   ├── todo-job/k8s/          # CronJob manifests
│   └── kustomization.yaml     # Base Kustomize configuration
├── overlays/
│   ├── staging/               # Staging environment overlays
│   │   ├── kustomization.yaml # Staging patches and image tags
│   │   ├── broadcaster-patch.yaml
│   │   ├── backend-nats-patch.yaml
│   │   ├── postgres-patch.yaml
│   │   └── todo-backend-patch.yaml
│   └── prod/                  # Production environment overlays
│       └── kustomization.yaml # Production patches and image tags
├── argocd-staging.yaml        # ArgoCD Application for staging
├── argocd-production.yaml      # ArgoCD Application for production
└── key.txt                     # SSH key for ArgoCD repository access
```

## 🏗️ Services Overview

### Base Manifests

Each service has its own base Kubernetes configuration in `base/`:

#### Broadcaster (`base/broadcaster/k8s/`)

- **Deployment:** Listens to NATS events and sends Discord notifications
- **Secret:** Stores Discord webhook URL
- **Kustomization:** Image configuration

#### Todo App (`base/todo-app/k8s/`)

- **Deployment:** Node.js web server with frontend
- **Kustomization:** Image configuration

#### Todo Backend (`base/todo-backend/k8s/`)

- **Deployment:** Node.js REST API
- **StatefulSet:** PostgreSQL 16 database
- **Service:** Internal networking
- **CronJob:** Backup job for database
- **Secrets:** Database credentials (encrypted)
- **Kustomization:** Image and configuration

#### Todo Job (`base/todo-job/k8s/`)

- **CronJob:** Hourly Wikipedia article fetcher
- **Kustomization:** Image configuration

### Environment-Specific Overlays

Kustomize overlays apply environment-specific patches:

#### Staging (`overlays/staging/`)

- Lower resource limits for cost savings
- Empty Discord webhook (logs instead of posting)
- PostgreSQL reduced resources
- Backend NATS configuration
- Namespace: `todo-staging`

#### Production (`overlays/prod/`)

- Production resource limits
- Full Discord webhook configuration
- PostgreSQL production settings
- Full NATS integration
- Namespace: `todo`

## 🔄 GitOps Workflow

### How It Works

1. **Code Changes:** Developer pushes to [dwk-project-code](https://github.com/Zanaad/dwk-project-code)
2. **CI/CD Pipeline:** GitHub Actions builds and pushes Docker images
3. **GitOps Update:** Workflow updates image tags in this repository
4. **ArgoCD Detection:** ArgoCD detects manifest changes
5. **Automatic Sync:** ArgoCD applies changes to the cluster

### Image Tag Updates

The CI/CD pipeline automatically updates image tags:

- **Staging:** `overlays/staging/kustomization.yaml` - tagged as `staging-{commit-sha}`
- **Production:** `overlays/prod/kustomization.yaml` - tagged as `v{major}.{minor}.{patch}`

## 📋 ArgoCD Applications

### Staging Application (`argocd-staging.yaml`)

```yaml
Application: todo-staging
Path: overlays/staging
Namespace: todo-staging
Sync Policy: Automatic with prune and self-heal
```

Deploy to cluster:

```bash
kubectl apply -f argocd-staging.yaml
```

### Production Application (`argocd-production.yaml`)

```yaml
Application: todo-production
Path: overlays/prod
Namespace: todo
Sync Policy: Automatic with prune and self-heal
```

Deploy to cluster:

```bash
kubectl apply -f argocd-production.yaml
```

## 🔐 Secrets Management

### Encrypted Secrets (SOPS)

Some secrets are encrypted with SOPS:

- `base/broadcaster/k8s/secret.enc.yaml` - Discord webhook
- `base/todo-backend/k8s/secret.enc.yaml` - Database credentials

## 🚀 Deployment Commands

### Apply Staging

```bash
# Using kubectl + Kustomize
kubectl apply -k overlays/staging

# Or let ArgoCD handle it
kubectl apply -f argocd-staging.yaml
```

### Apply Production

```bash
# Using kubectl + Kustomize
kubectl apply -k overlays/prod

# Or let ArgoCD handle it
kubectl apply -f argocd-production.yaml
```

## 📝 Kustomize Patterns

### Base Configuration

The `base/kustomization.yaml` includes all services:

```yaml
resources:
  - todo-backend/k8s
  - todo-app/k8s
  - todo-job/k8s
  - broadcaster/k8s
```

### Overlay Patches

Staging overlays modify:

- **Images:** Update to staging tags
- **Resources:** Reduce CPU/memory limits
- **ConfigMaps:** Environment-specific settings
- **Secrets:** Staging credentials
