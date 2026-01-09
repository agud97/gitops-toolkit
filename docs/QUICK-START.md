# 🚀 Quick Start Guide

Это руководство поможет быстро начать использовать GitOps Toolkit.

## Предварительные требования

- Kubernetes кластер (1.28+)
- kubectl настроен для подключения к кластеру
- Helm 3.x
- ArgoCD CLI (опционально)

## Шаг 1: Fork/Clone репозитория

```bash
# Вариант 1: Fork на GitHub и clone
git clone https://github.com/YOUR_USERNAME/gitops-toolkit.git
cd gitops-toolkit

# Вариант 2: Просто clone (для личного использования)
git clone https://github.com/ORIGINAL_OWNER/gitops-toolkit.git
cd gitops-toolkit
```

## Шаг 2: Установка ArgoCD

```bash
# Создание namespace
kubectl create namespace argocd

# Установка ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Ожидание готовности
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Получение пароля admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

# Port-forward для доступа к UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Откройте https://localhost:8080 и войдите с логином `admin`.

## Шаг 3: Настройка репозитория

### Вариант A: Публичный репозиторий

Никаких дополнительных действий не требуется.

### Вариант B: Приватный репозиторий

```bash
# Через CLI
argocd login localhost:8080 --username admin --password <password> --insecure

argocd repo add https://github.com/YOUR_USERNAME/gitops-toolkit.git \
  --username <github-username> \
  --password <github-token>
```

Или через UI: Settings → Repositories → Connect Repo

## Шаг 4: Редактирование конфигурации

### 4.1 Обновите URL репозитория

Замените `YOUR_USERNAME` на ваш GitHub username во всех файлах:

```bash
# Linux/Mac
find . -name "*.yaml" -exec sed -i 's|YOUR_USERNAME|your-actual-username|g' {} \;

# Или вручную отредактируйте:
# - bootstrap/app-of-apps.yaml
# - applications/*/application.yaml (где есть ссылки на репозиторий)
```

### 4.2 Выберите нужные компоненты

По умолчанию все приложения включены. Чтобы отключить ненужные:

```bash
# Переименуйте файл, добавив суффикс -disabled
mv applications/centrifugo/application.yaml applications/centrifugo/application.yaml-disabled
```

## Шаг 5: Применение AppProject и App-of-Apps

```bash
# Создание AppProject
kubectl apply -f bootstrap/project.yaml

# Применение root application
kubectl apply -f bootstrap/app-of-apps.yaml
```

## Шаг 6: Мониторинг синхронизации

```bash
# Через CLI
argocd app list
argocd app get gitops-toolkit-apps

# Или в UI
# https://localhost:8080/applications
```

## Шаг 7: Post-installation задачи

### Vault

После установки Vault требуется инициализация:

```bash
# Инициализация
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 -key-threshold=1 -format=json > vault-keys.json

# ⚠️ Сохраните vault-keys.json в безопасном месте!

# Unseal
UNSEAL_KEY=$(cat vault-keys.json | jq -r '.unseal_keys_b64[0]')
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### Grafana

```bash
# Получение пароля
kubectl get secret -n monitoring victoria-metrics-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d

# Port-forward
kubectl port-forward svc/victoria-metrics-grafana -n monitoring 3000:80
```

### PostgreSQL

```bash
# Проверка кластера
kubectl get clusters -n cnpg-system

# Получение connection string
kubectl get secret pg-cluster-app -n cnpg-system \
  -o jsonpath='{.data.uri}' | base64 -d
```

## Типичные сценарии использования

### Сценарий 1: Базовый мониторинг

Установите только:
- ingress-nginx
- cert-manager
- victoria-metrics

```bash
# Отключите остальные
for app in vault kafka kong postgresql keycloak redis centrifugo victoria-logs; do
  mv applications/$app/application.yaml applications/$app/application.yaml-disabled 2>/dev/null
done
```

### Сценарий 2: Полный стек для микросервисов

Установите всё:
- ingress-nginx (входящий трафик)
- cert-manager (TLS)
- vault (секреты)
- kong (API gateway)
- kafka (events)
- postgresql (данные)
- redis (кэш)
- victoria-metrics (метрики)
- grafana (дашборды)

### Сценарий 3: Только базы данных

```bash
# Оставьте только
# - postgresql
# - redis

for app in vault kafka kong keycloak centrifugo victoria-metrics victoria-logs grafana; do
  mv applications/$app/application.yaml applications/$app/application.yaml-disabled 2>/dev/null
done
```

## Кастомизация для нового проекта

1. Fork этого репозитория
2. Создайте новую ветку: `git checkout -b project/my-new-project`
3. Отредактируйте `helm-values/*/values-*.yaml` под ваши нужды
4. Обновите namespaces, domains, resources
5. Commit и push
6. В ArgoCD укажите вашу ветку как `targetRevision`

## Troubleshooting

### ArgoCD не видит репозиторий

```bash
argocd repo list
# Если пусто, добавьте репозиторий
argocd repo add https://github.com/... --username ... --password ...
```

### Application в статусе "Unknown"

```bash
argocd app get <app-name> --refresh
argocd app sync <app-name>
```

### Helm chart не найден

```bash
# Проверьте что helm repo доступен
helm repo add <repo-name> <repo-url>
helm repo update
helm search repo <chart-name>
```

### Pod не запускается

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Следующие шаги

1. Изучите [документацию](docs/) по каждому компоненту
2. Настройте [Vault](docs/VAULT.md) для управления секретами
3. Создайте [ServiceMonitors](docs/MONITORING.md) для ваших приложений
4. Настройте [Kong routes](docs/KONG.md) для API
