# 🚀 GitOps Toolkit - Reusable ArgoCD Applications

[![ArgoCD](https://img.shields.io/badge/ArgoCD-Ready-blue)](https://argo-cd.readthedocs.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-brightgreen)](https://kubernetes.io/)

Набор готовых к использованию ArgoCD Applications для быстрого развёртывания инфраструктурных сервисов в Kubernetes.

## 📋 Содержание

- [Быстрый старт](#-быстрый-старт)
- [Структура проекта](#-структура-проекта)
- [Доступные компоненты](#-доступные-компоненты)
- [Использование](#-использование)
- [Кастомизация](#-кастомизация)

## ⚡ Быстрый старт

```bash
# 1. Форкните репозиторий или клонируйте
git clone https://github.com/YOUR_USERNAME/gitops-toolkit.git
cd gitops-toolkit

# 2. Установите ArgoCD (если ещё не установлен)
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Добавьте репозиторий в ArgoCD
argocd repo add https://github.com/YOUR_USERNAME/gitops-toolkit.git

# 4. Выберите нужные приложения и примените
kubectl apply -f bootstrap/app-of-apps.yaml
# или выборочно:
kubectl apply -f applications/vault/application.yaml
```

## 📁 Структура проекта

```
gitops-toolkit/
├── README.md
├── bootstrap/                    # Начальная загрузка
│   ├── app-of-apps.yaml         # Root Application (App-of-Apps паттерн)
│   └── project.yaml             # AppProject
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
├── helm-values/                  # Helm values для каждого сервиса
│   ├── vault/
│   │   ├── values-base.yaml     # Базовая конфигурация
│   │   ├── values-dev.yaml      # Development overrides
│   │   └── values-prod.yaml     # Production overrides
│   └── ...
│
├── manifests/                    # Raw Kubernetes манифесты
│   ├── operators/               # CRDs и операторы
│   └── custom/                  # Кастомные ресурсы
│
├── overlays/                     # Kustomize overlays для разных окружений
│   ├── dev/
│   ├── staging/
│   └── prod/
│
└── docs/                         # Документация
    ├── VAULT.md
    ├── KAFKA.md
    └── ...
```

## 🧩 Доступные компоненты

| Компонент | Версия | Описание | Статус |
|-----------|--------|----------|--------|
| **Security & Secrets** |
| [HashiCorp Vault](applications/vault/) | 1.20.x | Управление секретами | ✅ Ready |
| [Vault Secrets Operator](applications/vault-secrets-operator/) | 0.10.x | Синхронизация секретов в K8s | ✅ Ready |
| [Cert-Manager](applications/cert-manager/) | 1.16.x | Управление сертификатами | ✅ Ready |
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

## 🔧 Использование

### Вариант 1: App-of-Apps (рекомендуется)

Используйте паттерн App-of-Apps для управления всеми приложениями:

```yaml
# Отредактируйте bootstrap/app-of-apps.yaml
# Укажите ваш репозиторий и выберите нужные приложения
kubectl apply -f bootstrap/app-of-apps.yaml
```

### Вариант 2: Выборочная установка

Установите только нужные компоненты:

```bash
# Vault
kubectl apply -f applications/vault/application.yaml

# Kafka
kubectl apply -f applications/kafka/operator.yaml
kubectl apply -f applications/kafka/cluster.yaml

# Victoria Metrics
kubectl apply -f applications/victoria-metrics/application.yaml
```

### Вариант 3: Копирование в свой проект

Скопируйте нужные директории в ваш GitOps репозиторий:

```bash
cp -r applications/vault/ /path/to/your/gitops-repo/apps/
cp -r helm-values/vault/ /path/to/your/gitops-repo/values/
```

## ⚙️ Кастомизация

### Переменные окружения

Каждый компонент поддерживает кастомизацию через:

1. **Helm Values** - `helm-values/<component>/values-<env>.yaml`
2. **Kustomize Overlays** - `overlays/<env>/`
3. **ArgoCD Application параметры** - прямо в Application spec

### Пример кастомизации Vault:

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

## 🔒 Безопасность

⚠️ **Важно:**
- Не храните секреты в Git
- Используйте External Secrets Operator или VSO для синхронизации секретов
- Сканируйте репозиторий на наличие утечек (gitleaks, trufflehog)

## 📚 Документация

- [Vault Setup Guide](docs/VAULT.md)
- [Kafka Configuration](docs/KAFKA.md)
- [Kong Routes](docs/KONG.md)
- [Monitoring Stack](docs/MONITORING.md)

## 🤝 Contributing

1. Fork репозитория
2. Создайте feature branch
3. Commit изменения
4. Push и создайте Pull Request

## 📄 License

MIT License - используйте свободно!
