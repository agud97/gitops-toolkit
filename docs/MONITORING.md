# Мониторинг и Observability - Руководство

## Обзор

Стек мониторинга включает:
- **VictoriaMetrics** - хранение метрик (Prometheus-compatible)
- **VMAgent** - сбор метрик
- **VMAlert** - алертинг
- **Grafana** - визуализация
- **Victoria Logs** - централизованные логи

## Архитектура

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

## Установка

### VictoriaMetrics Stack

```bash
kubectl apply -f applications/victoria-metrics/application.yaml
```

### Victoria Logs

```bash
kubectl apply -f applications/victoria-logs/application.yaml
```

### Отдельная Grafana (опционально)

```bash
kubectl apply -f applications/grafana/application.yaml
```

## Endpoints

| Компонент | URL | Назначение |
|-----------|-----|------------|
| VictoriaMetrics | `http://vmsingle-victoria-metrics.monitoring:8429` | Запросы метрик |
| VMAgent | `http://vmagent-victoria-metrics.monitoring:8429` | Прием метрик |
| Grafana | `http://grafana.monitoring:80` | UI дашборды |
| Victoria Logs | `http://victoria-logs.victoria-logs:9428` | Логи |

## Grafana

### Доступ

```bash
# Port-forward
kubectl port-forward svc/victoria-metrics-grafana -n monitoring 3000:80

# URL: http://localhost:3000
# User: admin
# Password: получить из secret
kubectl get secret -n monitoring victoria-metrics-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

### Добавление Datasource

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

### Dashboard через ConfigMap

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

Для мониторинга своих приложений создайте ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-metrics
  namespace: monitoring
  labels:
    release: victoria-metrics  # Должен совпадать с selector VMAgent
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

Для подов без Service:

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

### Создание правила

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

### Запросы логов

```bash
# Query через API
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=kubernetes_namespace_name:app AND error'

# С лимитом
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=kubernetes_namespace_name:app' \
  --data-urlencode 'limit=100'
```

### LogsQL примеры

```
# Все логи из namespace
kubernetes_namespace_name:app

# Ошибки
kubernetes_namespace_name:app AND _msg:error

# По поду
kubernetes_pod_name:my-app-*

# За последний час
kubernetes_namespace_name:app AND _time:1h

# С регулярным выражением
kubernetes_namespace_name:app AND _msg:~"error|exception|failed"
```

## PromQL Примеры

### CPU

```promql
# CPU usage по поду
rate(container_cpu_usage_seconds_total{namespace="app"}[5m])

# CPU usage в процентах от request
sum(rate(container_cpu_usage_seconds_total{namespace="app"}[5m])) by (pod)
/ sum(kube_pod_container_resource_requests{resource="cpu", namespace="app"}) by (pod)
* 100
```

### Memory

```promql
# Memory usage
container_memory_working_set_bytes{namespace="app"}

# Memory usage в процентах от limit
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

### Метрики не появляются

```bash
# Проверка targets в VMAgent
kubectl port-forward svc/vmagent-victoria-metrics -n monitoring 8429:8429
# Откройте http://localhost:8429/targets

# Проверка что приложение отдает метрики
kubectl port-forward svc/my-app -n app 8080:8080
curl http://localhost:8080/metrics

# Логи VMAgent
kubectl logs -n monitoring -l app.kubernetes.io/name=vmagent
```

### Grafana не показывает данные

```bash
# Проверка datasource
kubectl exec -it -n monitoring deploy/grafana -- \
  curl -s http://vmsingle-victoria-metrics:8429/api/v1/query?query=up

# Проверка что метрики есть
curl -G 'http://localhost:8429/api/v1/query' \
  --data-urlencode 'query=up'
```

### Victoria Logs не собирает логи

```bash
# Проверка Fluent Bit
kubectl logs -n victoria-logs -l app.kubernetes.io/name=fluent-bit

# Проверка что логи доступны
curl -G 'http://victoria-logs:9428/select/logsql/query' \
  --data-urlencode 'query=*' \
  --data-urlencode 'limit=10'
```
