## 🚀 GitOps Toolkit - Reusable ArgoCD Applications

[![ArgoCD](https://img.shields.io/badge/ArgoCD-Ready-blue)](https://argo-cd.readthedocs.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-brightgreen)](https://kubernetes.io/)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A collection of production-ready ArgoCD Applications for rapid deployment of infrastructure services in Kubernetes.

## 📋 Table of Contents

- [Quick Start](#-quick-start)
- [Project Structure](#-project-structure)
- [Available Components](#-available-components)
- [Usage](#-usage)
- [Customization](#-customization)

## ⚡ Quick Start

```bash
# 1. Fork or clone the repository
git clone https://github.com/agud97/gitops-toolkit.git
cd gitops-toolkit

# 2. Install ArgoCD (if not already installed)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Add repository to ArgoCD
argocd repo add https://github.com/agud97/gitops-toolkit.git

# 4. Apply the applications you need
kubectl apply -f bootstrap/app-of-apps.yaml
# or selectively:
kubectl apply -f applications/vault/application.yaml
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

## ⚙️ Customization

### Environment Variables

Each component supports customization through:

1. **Helm Values** - `helm-values/<component>/values-<env>.yaml`
2. **Kustomize Overlays** - `overlays/<env>/`
3. **ArgoCD Application parameters** - directly in Application spec

### Example: Customizing Vault

```yaml
# helm-values/vault/values-prod.yaml
server:
  ha:
    enabled: true
    replicas: 3
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
  dataStorage:
    size: 50Gi
    storageClass: fast-ssd
```

## 🔒 Security

⚠️ **Important:**
- Never store secrets in Git
- Use External Secrets Operator or VSO for secrets synchronization
- Scan repository for leaks (gitleaks, trufflehog)

## 📚 Documentation

- [Vault Setup Guide](docs/VAULT.md)
- [Kafka Configuration](docs/KAFKA.md)
- [Kong Routes](docs/KONG.md)
- [Monitoring Stack](docs/MONITORING.md)
- [Quick Start Guide](docs/QUICK-START.md)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push and create a Pull Request

## 📄 License

MIT License - feel free to use!
