# HashiCorp Vault - Setup Guide

## Overview

HashiCorp Vault is a centralized solution for secrets management, data encryption, and access control.

## Architecture

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

## Installation

### 1. Apply ArgoCD Application

```bash
kubectl apply -f applications/vault/application.yaml
```

### 2. Initialize Vault (Manual Operation)

After the pod starts, you need to initialize Vault:

```bash
# Check status
kubectl exec -n vault vault-0 -- vault status

# Initialize (run ONCE only!)
kubectl exec -n vault vault-0 -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > vault-init.json

# ⚠️ IMPORTANT: Store vault-init.json securely!
# Contains unseal keys and root token
```

### 3. Unseal Vault

```bash
# Get unseal key from file
UNSEAL_KEY=$(cat vault-init.json | jq -r '.unseal_keys_b64[0]')

# Unseal
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### 4. Configure Kubernetes Auth

```bash
ROOT_TOKEN=$(cat vault-init.json | jq -r '.root_token')

# Login
kubectl exec -n vault vault-0 -- vault login $ROOT_TOKEN

# Enable Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Configure Kubernetes auth
kubectl exec -n vault vault-0 -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'

# Enable KV-v2 secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2
```

### 5. Create Policy and Role

```bash
# Create policy
kubectl exec -n vault vault-0 -- sh -c 'echo '\''
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
'\'' | vault policy write app-secrets -'

# Create role for namespace
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/app-role \
    bound_service_account_names=default \
    bound_service_account_namespaces=default,app \
    policies=app-secrets \
    ttl=1h
```

## Using with VSO (Vault Secrets Operator)

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

### VaultDynamicSecret (for database credentials)

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

## Management Commands

```bash
# Vault status
kubectl exec -n vault vault-0 -- vault status

# List secrets
kubectl exec -n vault vault-0 -- vault kv list secret/

# Read secret
kubectl exec -n vault vault-0 -- vault kv get secret/path/to/secret

# Write secret
kubectl exec -n vault vault-0 -- vault kv put secret/path/to/secret \
  username="user" \
  password="pass"

# Delete secret
kubectl exec -n vault vault-0 -- vault kv delete secret/path/to/secret
```

## Auto-Unseal

For production, it's recommended to configure auto-unseal using:

- AWS KMS
- GCP Cloud KMS
- Azure Key Vault
- HashiCorp Cloud Platform

Example for AWS KMS:

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

### Vault sealed after restart

```bash
kubectl exec -n vault vault-0 -- vault operator unseal $UNSEAL_KEY
```

### VSO not syncing secrets

```bash
# Check events
kubectl get events -n app --sort-by='.lastTimestamp' | grep -i vault

# Check VaultAuth
kubectl describe vaultauth -n app

# VSO logs
kubectl logs -n vault-secrets-operator -l app.kubernetes.io/name=vault-secrets-operator
```
