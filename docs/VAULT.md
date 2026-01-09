# HashiCorp Vault - Руководство

## Обзор

HashiCorp Vault - централизованное решение для управления секретами, шифрования данных и управления доступом.

## Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                     Vault Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────────────────────┐ │
│  │   vault (ns)     │     │   vault-secrets-operator (ns)    │ │
│  │                  │     │                                   │ │
│  │  ┌────────────┐  │     │  ┌─────────────────────────────┐ │ │
│  │  │  vault-0   │◄─┼─────┼──│  VSO Controller             │ │ │
│  │  │            │  │     │  │                             │ │ │
│  │  │ - KV-v2    │  │     │  │  Watches:                   │ │ │
│  │  │ - K8s Auth │  │     │  │  - VaultStaticSecret        │ │ │
│  │  └────────────┘  │     │  │  - VaultDynamicSecret       │ │ │
│  │                  │     │  └─────────────────────────────┘ │ │
│  └──────────────────┘     └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Установка

### 1. Применение ArgoCD Application

```bash
kubectl apply -f applications/vault/application.yaml
```

### 2. Инициализация Vault (ручная операция)

После того как pod запустится, необходимо инициализировать Vault:

```bash
# Проверка статуса
kubectl exec -n vault vault-0 -- vault status

# Инициализация (выполняется ОДИН раз!)
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > vault-init.json

# ⚠️ ВАЖНО: Сохраните vault-init.json в безопасном месте!
# Содержит unseal keys и root token
```

### 3. Разблокировка (Unseal)

```bash
# Получение unseal key из файла
UNSEAL_KEY=$(cat vault-init.json | jq -r '.unseal_keys_b64[0]')

# Разблокировка
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### 4. Настройка Kubernetes Auth

```bash
ROOT_TOKEN=$(cat vault-init.json | jq -r '.root_token')

# Авторизация
kubectl exec -n vault vault-0 -- vault login $ROOT_TOKEN

# Включение Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Настройка Kubernetes auth
kubectl exec -n vault vault-0 -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'

# Включение KV-v2 secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2
```

### 5. Создание политики и роли

```bash
# Создание политики
kubectl exec -n vault vault-0 -- sh -c 'echo '\''
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
'\'' | vault policy write app-secrets -'

# Создание роли для namespace
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/app-role \
    bound_service_account_names=default \
    bound_service_account_namespaces=default,app \
    policies=app-secrets \
    ttl=1h
```

## Использование с VSO (Vault Secrets Operator)

### VaultStaticSecret

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: my-secret
  namespace: app
spec:
  type: kv-v2
  mount: secret
  path: app/database

  vaultAuthRef: vault-auth

  destination:
    name: database-credentials
    create: true
    type: Opaque

  refreshAfter: 30s
```

### VaultDynamicSecret (для database credentials)

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: postgres-creds
  namespace: app
spec:
  mount: database
  path: creds/readonly

  vaultAuthRef: vault-auth

  destination:
    name: postgres-credentials
    create: true

  renewalPercent: 67
```

## Команды управления

```bash
# Статус Vault
kubectl exec -n vault vault-0 -- vault status

# Список секретов
kubectl exec -n vault vault-0 -- vault kv list secret/

# Чтение секрета
kubectl exec -n vault vault-0 -- vault kv get secret/path/to/secret

# Запись секрета
kubectl exec -n vault vault-0 -- vault kv put secret/path/to/secret \
  username="user" \
  password="pass"

# Удаление секрета
kubectl exec -n vault vault-0 -- vault kv delete secret/path/to/secret
```

## Auto-Unseal

Для production рекомендуется настроить auto-unseal через:

- AWS KMS
- GCP Cloud KMS
- Azure Key Vault
- HashiCorp Cloud Platform

Пример для AWS KMS:

```yaml
# helm values
server:
  standalone:
    config: |
      seal "awskms" {
        region     = "us-east-1"
        kms_key_id = "your-kms-key-id"
      }
```

## Troubleshooting

### Vault sealed после перезапуска

```bash
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### VSO не синхронизирует секреты

```bash
# Проверка событий
kubectl get events -n app --sort-by='.lastTimestamp' | grep -i vault

# Проверка VaultAuth
kubectl describe vaultauth -n app

# Логи VSO
kubectl logs -n vault-secrets-operator -l app.kubernetes.io/name=vault-secrets-operator
```
