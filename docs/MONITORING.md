# Monitoring & Observability - Setup Guide

## Overview

The monitoring stack includes:
- **VictoriaMetrics** - metrics storage (Prometheus-compatible)
- **VMAgent** - metrics collection
- **VMAlert** - alerting
- **Grafana** - visualization
- **Victoria Logs** - centralized logging

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Observability Stack                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────┐     ┌────────────────────┐                      │
│  │    Applications    │     │   Kubernetes       │                      │
│  │    (Pods)          │     │   Components       │                      │
│  └─────────┬──────────┘     └─────────┬──────────┘                      │
│            │ /metrics                  │ /metrics                        │
│            ▼                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐               │
│  │                    VMAgent                           │               │
│  │              (Scrapes metrics)                       │               │
│  └─────────────────────────┬───────────────────────────┘               │
│                            │                                             │
│                            ▼                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │               VictoriaMetrics                        │               │
│  │            (Time-series storage)                     │               │
│  └─────────────────────────┬───────────────────────────┘               │
│                            │                                             │
│            ┌───────────────┼───────────────┐                            │
│            ▼               ▼               ▼                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                    │
│  │   Grafana    │ │   VMAlert    │ │   API/CLI    │                    │
│  │  (Dashboards)│ │  (Alerting)  │ │  (PromQL)    │                    │
│  └──────────────┘ └──────────────┘ └──────────────┘                    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────┐               │
│  │                  Victoria Logs                       │               │
│  │              (Centralized logging)                   │               │
│  └─────────────────────────────────────────────────────┘               │
│                            ▲                                             │
│                            │                                             │
│  ┌─────────────────────────────────────────────────────┐               │
│  │                    Fluent Bit                        │               │
│  │              (Log collection)                        │               │
│  └─────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

## Installation

### VictoriaMetrics Stack

```bash
kubectl apply -f applications/victoria-metrics/application.yaml
```

### Victoria Logs

```bash
kubectl apply -f applications/victoria-logs/application.yaml
```

### Standalone Grafana (optional)

```bash
kubectl apply -f applications/grafana/application.yaml
```

## Endpoints

| Component | URL | Purpose |
|-----------|-----|---------|
| VictoriaMetrics | `http://vmsingle-victoria-metrics.monitoring:8429` | Metrics queries |
| VMAgent | `http://vmagent-victoria-metrics.monitoring:8429` | Metrics ingestion |
| Grafana | `http://grafana.monitoring:80` | UI dashboards |
| Victoria Logs | `http://victoria-logs.victoria-logs:9428` | Logs |

## Grafana

### Access

```bash
# Port-forward
kubectl port-forward svc/victoria-metrics-grafana -n monitoring 3000:80

# URL: http://localhost:3000
# User: admin
# Password: get from secret
kubectl get secret -n monitoring victoria-metrics-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

### Adding Datasource

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: VictoriaMetrics
        type: prometheus
        url: http://vmsingle-victoria-metrics-k8s-stack.monitoring:8429
        access: proxy
        isDefault: true
```

### Dashboard via ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  my-dashboard.json: |
    {
      "title": "My Dashboard",
      "panels": [...]
    }
```

## ServiceMonitor

To monitor your applications, create a ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-metrics
  namespace: monitoring
  labels:
    release: victoria-metrics  # Must match VMAgent selector
spec:
  namespaceSelector:
    matchNames:
      - app
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

## PodMonitor

For pods without a Service:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-pod-metrics
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - app
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

## VMAlert Rules

### Creating Rules

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: my-alerts
  namespace: monitoring
spec:
  groups:
    - name: my-app
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) 
            / sum(rate(http_requests_total[5m])) > 0.1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate detected"
            description: "Error rate is {{ $value | humanizePercentage }}"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="false"} == 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} not ready"
```

## Victoria Logs

### Querying Logs

```bash
# Query via API
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=kubernetes_namespace_name:app AND error'

# With limit
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=kubernetes_namespace_name:app' \
  --data-urlencode 'limit=100'
```

### LogsQL Examples

```
# All logs from namespace
kubernetes_namespace_name:app

# Errors
kubernetes_namespace_name:app AND _msg:error

# By pod
kubernetes_pod_name:my-app-*

# Last hour
kubernetes_namespace_name:app AND _time:1h

# With regex
kubernetes_namespace_name:app AND _msg:~"error|exception|failed"
```

## PromQL Examples

### CPU

```promql
# CPU usage by pod
rate(container_cpu_usage_seconds_total{namespace="app"}[5m])

# CPU usage as percentage of request
sum(rate(container_cpu_usage_seconds_total{namespace="app"}[5m])) by (pod)
/ sum(kube_pod_container_resource_requests{resource="cpu", namespace="app"}) by (pod)
* 100
```

### Memory

```promql
# Memory usage
container_memory_working_set_bytes{namespace="app"}

# Memory usage as percentage of limit
sum(container_memory_working_set_bytes{namespace="app"}) by (pod)
/ sum(kube_pod_container_resource_limits{resource="memory", namespace="app"}) by (pod)
* 100
```

### HTTP Metrics

```promql
# Request rate
sum(rate(http_requests_total{namespace="app"}[5m])) by (service)

# Error rate
sum(rate(http_requests_total{status=~"5..", namespace="app"}[5m]))
/ sum(rate(http_requests_total{namespace="app"}[5m]))

# Latency p99
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket{namespace="app"}[5m])) by (le, service)
)
```

## Troubleshooting

### Metrics not appearing

```bash
# Check targets in VMAgent
kubectl port-forward svc/vmagent-victoria-metrics -n monitoring 8429:8429
# Open http://localhost:8429/targets

# Check that application exposes metrics
kubectl port-forward svc/my-app -n app 8080:8080
curl http://localhost:8080/metrics

# VMAgent logs
kubectl logs -n monitoring -l app.kubernetes.io/name=vmagent
```

### Grafana not showing data

```bash
# Check datasource
kubectl exec -it -n monitoring deploy/grafana -- \
  curl -s http://vmsingle-victoria-metrics:8429/api/v1/query?query=up

# Check that metrics exist
curl -G 'http://localhost:8429/api/v1/query' \
  --data-urlencode 'query=up'
```

### Victoria Logs not collecting

```bash
# Check Fluent Bit
kubectl logs -n victoria-logs -l app.kubernetes.io/name=fluent-bit

# Check that logs are available
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=*' \
  --data-urlencode 'limit=10'
```
