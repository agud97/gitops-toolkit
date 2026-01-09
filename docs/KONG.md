# Kong Gateway - Setup Guide

## Overview

Kong Gateway is deployed in DBless mode - configuration is stored in ConfigMap and managed through Git.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kong DBless Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      kong namespace                        │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              Kong Gateway (DBless)                   │  │  │
│  │  │                                                      │  │  │
│  │  │  ┌────────────────────────────────────────────────┐ │  │  │
│  │  │  │  ConfigMap: kong-declarative-config           │ │  │  │
│  │  │  │  (declarative_config: /kong_dbless/kong.yml)  │ │  │  │
│  │  │  └────────────────────────────────────────────────┘ │  │  │
│  │  │                        │                             │  │  │
│  │  │                        ▼                             │  │  │
│  │  │  ┌────────────────────────────────────────────────┐ │  │  │
│  │  │  │  Kong Proxy                                    │ │  │  │
│  │  │  │  - HTTP:  8000 → Service Port 80              │ │  │  │
│  │  │  │  - HTTPS: 8443 → Service Port 443             │ │  │  │
│  │  │  └────────────────────────────────────────────────┘ │  │  │
│  │  │                                                      │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                            │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

```bash
# Apply Application
kubectl apply -f applications/kong/application.yaml

# Apply route configuration
kubectl apply -f manifests/custom/kong-config/kong-config.yaml
```

## Declarative Configuration

### Basic Structure

```yaml
_format_version: "3.0"
_transform: true

services:
  - name: my-service
    host: backend.default.svc.cluster.local
    port: 8080
    protocol: http
    routes:
      - name: my-route
        paths:
          - /api/v1/myservice
        strip_path: true

plugins:
  - name: rate-limiting
    config:
      minute: 100
```

### Adding a New Service

1. Edit `manifests/custom/kong-config/kong-config.yaml`
2. Add the new service:

```yaml
services:
  # Existing services...
  
  # New service
  - name: new-service
    host: new-service.app.svc.cluster.local
    port: 8080
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    routes:
      - name: new-service-route
        paths:
          - /api/v1/new
        protocols:
          - http
          - https
        strip_path: true
        preserve_host: false
```

3. Commit and push to Git
4. ArgoCD will automatically sync changes

## Plugins

### Rate Limiting

```yaml
plugins:
  - name: rate-limiting
    service: my-service  # Or route, consumer
    config:
      minute: 100
      hour: 1000
      policy: local  # local, cluster, redis
      fault_tolerant: true
      hide_client_headers: false
```

### JWT Authentication

```yaml
plugins:
  - name: jwt
    service: my-service
    config:
      uri_param_names:
        - jwt
      header_names:
        - Authorization
      claims_to_verify:
        - exp

consumers:
  - username: my-consumer
    jwt_secrets:
      - key: my-jwt-key
        secret: your-secret-here
        algorithm: HS256
```

### CORS

```yaml
plugins:
  - name: cors
    service: my-service
    config:
      origins:
        - https://example.com
        - https://app.example.com
      methods:
        - GET
        - POST
        - PUT
        - DELETE
      headers:
        - Accept
        - Authorization
        - Content-Type
      exposed_headers:
        - X-Request-Id
      credentials: true
      max_age: 3600
```

### Request Transformer

```yaml
plugins:
  - name: request-transformer
    service: my-service
    config:
      add:
        headers:
          - "X-Custom-Header:value"
      remove:
        headers:
          - "X-Unwanted-Header"
```

### Logging

```yaml
plugins:
  - name: http-log
    config:
      http_endpoint: http://log-collector.logging.svc.cluster.local:8080/logs
      method: POST
      content_type: application/json
```

## Upstreams and Load Balancing

```yaml
upstreams:
  - name: my-upstream
    algorithm: round-robin  # round-robin, consistent-hashing, least-connections
    healthchecks:
      active:
        healthy:
          interval: 5
          successes: 2
        unhealthy:
          interval: 5
          http_failures: 3
          tcp_failures: 3
          timeouts: 3
        http_path: /health
        timeout: 5
      passive:
        healthy:
          successes: 5
        unhealthy:
          http_failures: 5
          tcp_failures: 5
          timeouts: 5
    targets:
      - target: backend-1.app.svc.cluster.local:8080
        weight: 100
      - target: backend-2.app.svc.cluster.local:8080
        weight: 100

services:
  - name: my-service
    host: my-upstream  # Reference to upstream
    port: 8080
```

## Management Commands

```bash
# Check Kong status
kubectl get pods -n kong

# Logs
kubectl logs -n kong -l app.kubernetes.io/name=kong

# Check configuration
kubectl get configmap -n kong

# Verify configuration is loaded
kubectl exec -it -n kong deploy/kong-kong -- kong config db_export

# Check routes
kubectl exec -it -n kong deploy/kong-kong -- kong config db_export | grep -A 10 "routes:"

# Health check
kubectl port-forward svc/kong-kong-proxy -n kong 8000:80
curl http://localhost:8000/status
```

## Troubleshooting

### Kong not starting

```bash
# Check events
kubectl get events -n kong --sort-by='.lastTimestamp'

# Check ConfigMap
kubectl get configmap -n kong kong-declarative-config -o yaml

# Validate configuration
kubectl exec -it -n kong deploy/kong-kong -- kong config parse /kong_dbless/kong.yml
```

### Route not working

```bash
# Check that configuration is loaded
kubectl exec -it -n kong deploy/kong-kong -- kong config db_export

# Test route
kubectl port-forward svc/kong-kong-proxy -n kong 8000:80
curl -v http://localhost:8000/api/v1/myservice

# Check logs
kubectl logs -n kong -l app.kubernetes.io/name=kong --tail=100
```

### Upstream unavailable

```bash
# Check that backend is accessible from Kong pod
kubectl exec -it -n kong deploy/kong-kong -- \
  curl -v http://backend.app.svc.cluster.local:8080/health
```
