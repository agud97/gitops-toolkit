# GitOps Toolkit - Reusable ArgoCD Applications

Production-ready ArgoCD Applications for rapid deployment of infrastructure services in Kubernetes.

## Table of Contents

- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Available Components](#available-components)
- [Usage](#usage)
- [Integration Examples](#integration-examples)
- [Customization](#customization)

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/agud97/gitops-toolkit.git
cd gitops-toolkit

# 2. Install ArgoCD (if not already installed)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Add repository to ArgoCD
argocd repo add https://github.com/agud97/gitops-toolkit.git

# 4. Apply the bootstrap
kubectl apply -f bootstrap/project.yaml
kubectl apply -f bootstrap/app-of-apps.yaml
```

## 📁 Project Structure

```
gitops-toolkit/
├── README.md
├── bootstrap/                    # Bootstrap configuration
│   ├── app-of-apps.yaml         # Root Application (App-of-Apps pattern)
│   └── project.yaml             # AppProject definition
│
├── applications/                 # ArgoCD Applications
│   ├── vault/                   # HashiCorp Vault
│   ├── kafka/                   # Apache Kafka (Strimzi)
│   ├── kong/                    # Kong API Gateway
│   ├── victoria-metrics/        # VictoriaMetrics Stack
│   ├── postgresql/              # CloudNativePG PostgreSQL
│   ├── keycloak/                # Keycloak IAM
│   ├── ingress-nginx/           # Ingress NGINX Controller
│   ├── cert-manager/            # Cert-Manager
│   ├── redis/                   # Redis/Valkey
│   ├── grafana/                 # Grafana
│   └── ...
│
├── helm-values/                  # Helm values for each service
│   ├── vault/
│   │   ├── values-base.yaml     # Base configuration
│   │   ├── values-dev.yaml      # Development overrides
│   │   └── values-prod.yaml     # Production overrides
│   └── ...
│
├── manifests/                    # Raw Kubernetes manifests
│   ├── operators/               # CRDs and operators
│   └── custom/                  # Custom resources
│
├── overlays/                     # Kustomize overlays for different environments
│   ├── dev/
│   ├── staging/
│   └── prod/
│
└── docs/                         # Documentation
    ├── VAULT.md
    ├── KAFKA.md
    └── ...
```

## 🧩 Available Components

| Component | Version | Description | Status |
|-----------|---------|-------------|--------|
| **Security & Secrets** |
| [HashiCorp Vault](applications/vault/) | 1.20.x | Secrets management | ✅ Ready |
| [Vault Secrets Operator](applications/vault-secrets-operator/) | 0.10.x | Sync secrets to K8s | ✅ Ready |
| [Cert-Manager](applications/cert-manager/) | 1.16.x | Certificate management | ✅ Ready |
| **API Gateway & Networking** |
| [Kong Gateway](applications/kong/) | 3.9.x | API Gateway (DBless/DB) | ✅ Ready |
| [Ingress NGINX](applications/ingress-nginx/) | 1.13.x | Ingress Controller | ✅ Ready |
| **Message Brokers** |
| [Apache Kafka](applications/kafka/) | 4.0.x | Message Broker (Strimzi) | ✅ Ready |
| **Databases** |
| [PostgreSQL](applications/postgresql/) | 17.x | PostgreSQL (CloudNativePG) | ✅ Ready |
| [Redis/Valkey](applications/redis/) | 8.x | In-memory Cache | ✅ Ready |
| **Monitoring & Observability** |
| [VictoriaMetrics](applications/victoria-metrics/) | 1.125.x | Metrics & Monitoring | ✅ Ready |
| [Grafana](applications/grafana/) | 11.x | Dashboards & Visualization | ✅ Ready |
| [Victoria Logs](applications/victoria-logs/) | 1.x | Log Aggregation | ✅ Ready |
| **Identity & Access** |
| [Keycloak](applications/keycloak/) | 26.x | IAM/SSO | ✅ Ready |
| **Misc** |
| [Centrifugo](applications/centrifugo/) | 6.x | WebSocket Server | ✅ Ready |

## 🔧 Usage

### Option 1: App-of-Apps (Recommended)

Use the App-of-Apps pattern to manage all applications:

```yaml
# Edit bootstrap/app-of-apps.yaml
# Specify your repository and select needed applications
kubectl apply -f bootstrap/app-of-apps.yaml
```

### Option 2: Selective Installation

Install only the components you need:

```bash
# Vault
kubectl apply -f applications/vault/application.yaml

# Kafka
kubectl apply -f applications/kafka/operator.yaml
kubectl apply -f applications/kafka/cluster.yaml

# Victoria Metrics
kubectl apply -f applications/victoria-metrics/application.yaml
```

### Option 3: Copy to Your Project

Copy the needed directories to your GitOps repository:

```bash
cp -r applications/vault/ /path/to/your/gitops-repo/apps/
cp -r helm-values/vault/ /path/to/your/gitops-repo/values/
```

## Integration Examples

### VSO + CloudNativePG (Database Secrets from Vault)

Sync database credentials from Vault to Kubernetes Secrets for CNPG managed roles:

```
manifests/custom/vso-cnpg-integration/
├── README.md                    # Setup instructions
├── vault-setup.yaml             # VaultConnection + VaultAuth
├── vso-cnpg-secrets.yaml        # VaultStaticSecret resources
└── pg-cluster-with-vso.yaml     # CNPG Cluster using VSO secrets
```

See [VSO-CNPG Integration Guide](manifests/custom/vso-cnpg-integration/README.md).

### Kong DBless Configuration

Declarative routing configuration for Kong Gateway:

```
manifests/custom/kong-config/
└── kong-config.yaml             # ConfigMap with routes
```

Example services: user-service, account-service, wallet-service, payment-service, admin-backend.

## Customization

### Multi-Source Applications

Applications use ArgoCD multi-source pattern for separate Helm values:

```yaml
sources:
  - repoURL: https://github.com/agud97/gitops-toolkit.git
    targetRevision: HEAD
    ref: values

  - repoURL: https://helm.releases.hashicorp.com
    chart: vault
    targetRevision: 0.30.0
    helm:
      valueFiles:
        - $values/helm-values/vault/values-base.yaml
        - $values/helm-values/vault/values-prod.yaml
```

### Environment-Specific Values

```
helm-values/<component>/
├── values-base.yaml    # Common settings
├── values-dev.yaml     # Development overrides
└── values-prod.yaml    # Production overrides
```

### Example: Production Vault

```yaml
# helm-values/vault/values-prod.yaml
server:
  ha:
    enabled: true
    replicas: 3
  dataStorage:
    size: 50Gi
    storageClass: fast-ssd
```

## Security

- Never store secrets in Git
- Use Vault Secrets Operator (VSO) or External Secrets Operator
- Placeholder secrets are marked with `CHANGE_ME` comments
- All sensitive domains use `example.com` placeholder

## Documentation

- [VSO-CNPG Integration](manifests/custom/vso-cnpg-integration/README.md)
- [Vault Setup Guide](docs/VAULT.md)
- [Kafka Configuration](docs/KAFKA.md)
- [Kong Routes](docs/KONG.md)
- [Monitoring Stack](docs/MONITORING.md)

## License

MIT License
