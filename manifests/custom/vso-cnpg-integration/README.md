# VSO + CloudNativePG Integration

This directory contains examples for integrating Vault Secrets Operator with CloudNativePG for automated database credential management.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     VSO → CNPG Integration                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐     ┌──────────────────────────────────┐  │
│  │   Vault (vault)  │     │   VSO (vault-secrets-operator)   │  │
│  │                  │     │                                   │  │
│  │  KV-v2 Engine    │────►│  VaultStaticSecret               │  │
│  │  secret/cnpg/*   │     │  - watches Vault paths            │  │
│  │                  │     │  - syncs to K8s Secrets           │  │
│  └──────────────────┘     └──────────────────────────────────┘  │
│                                         │                        │
│                                         ▼                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     cnpg-system namespace                  │   │
│  │                                                            │   │
│  │  K8s Secrets ◄──────── VaultStaticSecret                  │   │
│  │  (secret-keycloak,     (syncs from Vault)                 │   │
│  │   secret-kong, etc.)                                       │   │
│  │         │                                                  │   │
│  │         ▼                                                  │   │
│  │  CNPG Cluster                                              │   │
│  │  managed.roles[].passwordSecret ──► K8s Secrets           │   │
│  │                                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Setup Steps

### 1. Configure Vault

```bash
# Enable KV-v2 secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# Enable Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Configure Kubernetes auth
kubectl exec -n vault vault-0 -- sh -c 'vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'

# Create policy for CNPG secrets
kubectl exec -n vault vault-0 -- sh -c 'echo "
path \"secret/data/cnpg/*\" {
  capabilities = [\"read\"]
}
path \"secret/metadata/cnpg/*\" {
  capabilities = [\"read\", \"list\"]
}" | vault policy write cnpg-secrets -'

# Create role for cnpg-system namespace
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/cnpg-secrets \
    bound_service_account_names=default \
    bound_service_account_namespaces=cnpg-system \
    policies=cnpg-secrets \
    ttl=1h
```

### 2. Create secrets in Vault

```bash
# Create database user secrets
kubectl exec -n vault vault-0 -- vault kv put secret/cnpg/keycloak \
    username="keycloak" \
    password="$(openssl rand -base64 24)"

kubectl exec -n vault vault-0 -- vault kv put secret/cnpg/kong \
    username="kong" \
    password="$(openssl rand -base64 24)"

# ... repeat for other services
```

### 3. Apply VaultStaticSecret resources

```bash
kubectl apply -f vso-cnpg-secrets.yaml
```

### 4. Verify secrets are synced

```bash
kubectl get secrets -n cnpg-system | grep secret-
kubectl get secret secret-keycloak -n cnpg-system -o jsonpath='{.data.password}' | base64 -d
```

## Files

- `vault-setup.yaml` - VaultConnection and VaultAuth for cnpg-system namespace
- `vso-cnpg-secrets.yaml` - VaultStaticSecret resources for database users
- `pg-cluster-with-vso.yaml` - Example CNPG Cluster using VSO-managed secrets
