# Apache Kafka (Strimzi) - Setup Guide

## Overview

Apache Kafka is deployed via Strimzi Operator in KRaft mode (without ZooKeeper).

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kafka Architecture (Strimzi KRaft)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Strimzi Operator                            │   │
│  │  Watches: Kafka, KafkaTopic, KafkaUser, KafkaConnect    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           kafka-cluster (KRaft mode)                     │   │
│  │                                                          │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                │   │
│  │  │Combined-0│ │Combined-1│ │Combined-2│                │   │
│  │  │Controller│ │Controller│ │Controller│                │   │
│  │  │+ Broker  │ │+ Broker  │ │+ Broker  │                │   │
│  │  └──────────┘ └──────────┘ └──────────┘                │   │
│  │                                                          │   │
│  │  ┌───────────────────────────────────┐                  │   │
│  │  │        Entity Operator            │                  │   │
│  │  │  - Topic Operator                 │                  │   │
│  │  │  - User Operator                  │                  │   │
│  │  └───────────────────────────────────┘                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Installation

### 1. Install Strimzi Operator

```bash
kubectl apply -f applications/kafka/operator.yaml
```

### 2. Create Kafka Cluster

```bash
kubectl apply -f applications/kafka/cluster.yaml
```

### 3. Check Status

```bash
# Cluster status
kubectl get kafka -n kafka

# Pods
kubectl get pods -n kafka

# Wait for readiness
kubectl wait kafka/kafka-cluster --for=condition=Ready --timeout=300s -n kafka
```

## Bootstrap Servers

```
# Internal (from within the cluster)
Plain:  kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
TLS:    kafka-cluster-kafka-bootstrap.kafka.svc.cluster.local:9093
```

## Topic Management

### Create Topic

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 604800000      # 7 days
    segment.bytes: 1073741824    # 1GB
    cleanup.policy: delete
```

```bash
kubectl apply -f topic.yaml
```

### Via CLI

```bash
# List topics
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

# Describe topic
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic my-topic

# Create topic (imperative)
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic test-topic --partitions 3 --replication-factor 3
```

## User Management

### Create User with SCRAM-SHA-512

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      # Read access
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Read
          - Describe
        host: "*"
      # Write access
      - resource:
          type: topic
          name: my-topic
          patternType: literal
        operations:
          - Write
        host: "*"
      # Consumer group
      - resource:
          type: group
          name: my-group
          patternType: prefix
        operations:
          - Read
        host: "*"
```

### Get Credentials

```bash
# Password is stored in Secret
kubectl get secret my-user -n kafka -o jsonpath='{.data.password}' | base64 -d
```

## Kafka Connect

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect
  namespace: kafka
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  version: 3.9.0
  replicas: 1
  bootstrapServers: kafka-cluster-kafka-bootstrap:9092
  config:
    group.id: connect-cluster
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
```

## Monitoring

### Kafka Exporter Metrics

If `kafkaExporter` is enabled, metrics are available at:
```
http://kafka-cluster-kafka-exporter.kafka.svc.cluster.local:9404/metrics
```

### ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: kafka-metrics
  namespace: kafka
spec:
  selector:
    matchLabels:
      strimzi.io/cluster: kafka-cluster
      strimzi.io/kind: Kafka
  podMetricsEndpoints:
    - port: tcp-prometheus
      path: /metrics
```

## Testing

### Producer

```bash
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic
```

### Consumer

```bash
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --from-beginning
```

## Troubleshooting

### Check Logs

```bash
# Broker logs
kubectl logs kafka-cluster-combined-0 -n kafka

# Operator logs
kubectl logs -n kafka -l strimzi.io/kind=cluster-operator
```

### Consumer Lag

```bash
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group
```

### Partition Reassignment

```bash
kubectl exec -it kafka-cluster-combined-0 -n kafka -- \
  bin/kafka-reassign-partitions.sh \
  --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "0,1,2" \
  --generate
```
