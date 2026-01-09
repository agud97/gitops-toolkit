# 🚀 Quick Start Guide

This guide will help you quickly start using the GitOps Toolkit.

## Prerequisites

- Kubernetes cluster (1.28+)
- kubectl configured to connect to the cluster
- Helm 3.x
- ArgoCD CLI (optional)

## Step 1: Clone the Repository

```bash
git clone https://github.com/agud97/gitops-toolkit.git
cd gitops-toolkit
```

## Step 2: Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for readiness
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

# Port-forward for UI access
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 and login with username `admin`.

## Step 3: Configure Repository

### Option A: Public Repository

No additional actions required.

### Option B: Private Repository

```bash
# Via CLI
argocd login localhost:8080 --username admin --password <password> --insecure

argocd repo add https://github.com/agud97/gitops-toolkit.git \
  --username <github-username> \
  --password <github-token>
```

Or via UI: Settings → Repositories → Connect Repo

## Step 4: Apply AppProject and App-of-Apps

```bash
# Create AppProject
kubectl apply -f bootstrap/project.yaml

# Apply root application
kubectl apply -f bootstrap/app-of-apps.yaml
```

## Step 5: Monitor Synchronization

```bash
# Via CLI
argocd app list
argocd app get gitops-toolkit-apps

# Or in UI
# https://localhost:8080/applications
```

## Step 6: Post-Installation Tasks

### Vault

After Vault installation, initialization is required:

```bash
# Initialize
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 -key-threshold=1 -format=json > vault-keys.json

# ⚠️ Store vault-keys.json securely!

# Unseal
UNSEAL_KEY=$(cat vault-keys.json | jq -r '.unseal_keys_b64[0]')
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### Grafana

```bash
# Get password
kubectl get secret -n monitoring victoria-metrics-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d

# Port-forward
kubectl port-forward svc/victoria-metrics-grafana -n monitoring 3000:80
```

### PostgreSQL

```bash
# Check cluster
kubectl get clusters -n cnpg-system

# Get connection string
kubectl get secret pg-cluster-app -n cnpg-system \
  -o jsonpath='{.data.uri}' | base64 -d
```

## Common Use Cases

### Scenario 1: Basic Monitoring

Install only:
- ingress-nginx
- cert-manager
- victoria-metrics

```bash
# Disable the rest
for app in vault kafka kong postgresql keycloak redis centrifugo victoria-logs; do
  mv applications/$app/application.yaml applications/$app/application.yaml-disabled 2>/dev/null
done
```

### Scenario 2: Full Microservices Stack

Install everything:
- ingress-nginx (incoming traffic)
- cert-manager (TLS)
- vault (secrets)
- kong (API gateway)
- kafka (events)
- postgresql (data)
- redis (cache)
- victoria-metrics (metrics)
- grafana (dashboards)

### Scenario 3: Databases Only

```bash
# Keep only
# - postgresql
# - redis

for app in vault kafka kong keycloak centrifugo victoria-metrics victoria-logs grafana; do
  mv applications/$app/application.yaml applications/$app/application.yaml-disabled 2>/dev/null
done
```

## Customization for a New Project

1. Fork this repository
2. Create a new branch: `git checkout -b project/my-new-project`
3. Edit `helm-values/*/values-*.yaml` for your needs
4. Update namespaces, domains, resources
5. Commit and push
6. In ArgoCD, specify your branch as `targetRevision`

## Troubleshooting

### ArgoCD can't see the repository

```bash
argocd repo list
# If empty, add the repository
argocd repo add https://github.com/... --username ... --password ...
```

### Application in "Unknown" status

```bash
argocd app get <app-name> --refresh
argocd app sync <app-name>
```

### Helm chart not found

```bash
# Check that helm repo is available
helm repo add <repo-name> <repo-url>
helm repo update
helm search repo <chart-name>
```

### Pod not starting

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Next Steps

1. Study the [documentation](docs/) for each component
2. Configure [Vault](docs/VAULT.md) for secrets management
3. Create [ServiceMonitors](docs/MONITORING.md) for your applications
4. Configure [Kong routes](docs/KONG.md) for your API
