# Kafka Cluster (KRaft Mode)

Apache Kafka cluster running in KRaft mode using Strimzi operator - deployed via ArgoCD.

## Overview

This repository contains Kubernetes manifests for deploying a Kafka cluster using Strimzi's KafkaNodePool and Kafka custom resources. The cluster runs in KRaft mode, eliminating the need for ZooKeeper.

## Architecture

**KRaft Mode with Node Pools:**
- **Controllers** (3 replicas): Manage cluster metadata using Raft consensus
- **Brokers** (3 replicas): Handle client requests and data replication
- **No ZooKeeper**: Uses Kafka's native KRaft protocol for consensus

## Features

- ✅ **KRaft Mode**: No ZooKeeper dependency
- ✅ **Node Pools**: Separate controller and broker nodes
- ✅ **Multiple Listeners**: Internal (plain, TLS) and external (NodePort)
- ✅ **Entity Operators**: Topic and User management
- ✅ **Persistent Storage**: Data persistence with PVCs
- ✅ **GitOps Ready**: Managed via ArgoCD

## Prerequisites

- Kubernetes cluster (v1.19+)
- Strimzi Kafka Operator installed (v0.48.0+)
- Default StorageClass or specify storageClass in manifests
- ArgoCD (for GitOps deployment)

## Repository Structure

```
kraft-kafka-server/
├── namespace.yaml         # kraft-kafka namespace
├── kafka-cluster.yaml     # Kafka CR and KafkaNodePool CRs
└── README.md              # This file
```

## Configuration

### Kafka Cluster Specification

| Component | Replicas | Memory | CPU | Storage |
|-----------|----------|--------|-----|---------|
| Controllers | 3 | 512Mi-1Gi | 500m-1000m | 10Gi |
| Brokers | 3 | 1Gi-2Gi | 500m-1000m | 10Gi |

### Kafka Configuration

```yaml
version: 4.0.0
metadataVersion: 4.0-IV0
num.partitions: 3
default.replication.factor: 3
min.insync.replicas: 2
offsets.topic.replication.factor: 3
transaction.state.log.replication.factor: 3
transaction.state.log.min.isr: 2
```

### Listeners

| Listener | Port | Type | TLS | Description |
|----------|------|------|-----|-------------|
| plain | 9092 | internal | No | Internal plaintext |
| tls | 9093 | internal | Yes | Internal encrypted |
| external | 9094 | nodeport | No | External access |

## Installation

### Using ArgoCD (Recommended)

This cluster is designed to be deployed via ArgoCD:

```bash
# Deployed automatically via app-of-apps pattern
kubectl apply -f https://raw.githubusercontent.com/kadirsahan/argocd-apps/main/app-of-apps.yaml
```

### Manual Installation

```bash
# Clone the repository
git clone https://github.com/kadirsahan/kraft-kafka-server.git
cd kraft-kafka-server

# Create namespace
kubectl apply -f namespace.yaml

# Deploy Kafka cluster (requires Strimzi operator)
kubectl apply -f kafka-cluster.yaml
```

## Verify Installation

```bash
# Check namespace
kubectl get ns kraft-kafka

# Check Kafka cluster status
kubectl get kafka -n kraft-kafka

# Check node pools
kubectl get kafkanodepool -n kraft-kafka

# Check pods
kubectl get pods -n kraft-kafka

# Watch cluster creation
kubectl get kafka kraft-kafka-cluster -n kraft-kafka -w
```

Expected output when ready:
```
NAME                   DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   METADATA STATE   WARNINGS
kraft-kafka-cluster                                                   True    KRaft
```

## Accessing Kafka

### From Within Kubernetes

```bash
# Bootstrap server for internal access
kraft-kafka-cluster-kafka-bootstrap.kraft-kafka.svc.cluster.local:9092
```

### From Outside Kubernetes (NodePort)

```bash
# Get NodePort
kubectl get svc -n kraft-kafka | grep nodeport

# Access via NodePort (port 9094)
<node-ip>:9094
```

## Using Kafka

### Create a Topic

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kraft-kafka
  labels:
    strimzi.io/cluster: kraft-kafka-cluster
spec:
  partitions: 3
  replicas: 3
  config:
    retention.ms: 86400000  # 1 day
    segment.bytes: 1073741824  # 1GB
```

Or using kubectl:
```bash
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kraft-kafka
  labels:
    strimzi.io/cluster: kraft-kafka-cluster
spec:
  partitions: 3
  replicas: 3
EOF
```

### Test with Kafka Client

```bash
# Deploy test client pod
kubectl run kafka-client -ti --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm=true --restart=Never -n kraft-kafka -- /bin/bash

# Inside the pod:
# Produce messages
kafka-console-producer.sh --bootstrap-server kraft-kafka-cluster-kafka-bootstrap:9092 --topic my-topic

# Consume messages (in another terminal)
kubectl run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.48.0-kafka-4.0.0 --rm=true --restart=Never -n kraft-kafka -- \
  kafka-console-consumer.sh --bootstrap-server kraft-kafka-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```

## Monitoring

```bash
# Check cluster status
kubectl get kafka kraft-kafka-cluster -n kraft-kafka -o yaml

# Check controller pods
kubectl get pods -n kraft-kafka -l strimzi.io/pool-name=controllers

# Check broker pods
kubectl get pods -n kraft-kafka -l strimzi.io/pool-name=brokers

# Check entity operator
kubectl get pods -n kraft-kafka -l strimzi.io/name=kraft-kafka-cluster-entity-operator

# View logs
kubectl logs -n kraft-kafka kraft-kafka-cluster-brokers-0 -c kafka
```

## Scaling

### Scale Brokers

```bash
kubectl patch kafkanodepool brokers -n kraft-kafka --type='merge' -p '{"spec":{"replicas":5}}'
```

### Scale Controllers

```bash
kubectl patch kafkanodepool controllers -n kraft-kafka --type='merge' -p '{"spec":{"replicas":5}}'
```

## Storage Configuration

By default, the cluster uses the default StorageClass. To use a specific StorageClass:

1. Edit `kafka-cluster.yaml`
2. Uncomment and set the `class` field:
   ```yaml
   storage:
     type: jbod
     volumes:
     - id: 0
       type: persistent-claim
       size: 10Gi
       deleteClaim: false
       class: managed-nfs-server  # Your StorageClass
   ```
3. Apply changes

## Upgrading

### Upgrade Kafka Version

```bash
kubectl patch kafka kraft-kafka-cluster -n kraft-kafka --type='merge' -p '{"spec":{"kafka":{"version":"4.1.0"}}}'
```

**Note**: Follow Strimzi upgrade guidelines for version compatibility.

## Troubleshooting

### Pods Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n kraft-kafka

# Check operator logs
kubectl logs -n kraft-strimzi-operator deployment/strimzi-cluster-operator
```

### PVC Pending

```bash
# Check PVCs
kubectl get pvc -n kraft-kafka

# Check StorageClass
kubectl get storageclass

# If using custom StorageClass, ensure it exists and is set correctly
```

### Cluster Not Ready

```bash
# Check Kafka resource status
kubectl describe kafka kraft-kafka-cluster -n kraft-kafka

# Check conditions
kubectl get kafka kraft-kafka-cluster -n kraft-kafka -o jsonpath='{.status.conditions}'
```

## Uninstalling

```bash
# Delete Kafka cluster
kubectl delete kafka kraft-kafka-cluster -n kraft-kafka

# Delete node pools
kubectl delete kafkanodepool -n kraft-kafka --all

# Delete namespace
kubectl delete namespace kraft-kafka
```

**Warning**: This will delete all Kafka data. Ensure you have backups if needed.

## Related Repositories

- [strimzi-operator](https://github.com/kadirsahan/strimzi-operator) - Strimzi operator installation
- [argocd-gitops](https://github.com/kadirsahan/argocd-gitops) - ArgoCD installation
- [argocd-apps](https://github.com/kadirsahan/argocd-apps) - App-of-Apps GitOps repository
- [kafka-connect](https://github.com/kadirsahan/kafka-connect) - Kafka Connect deployment

## Documentation

- [Strimzi Documentation](https://strimzi.io/documentation/)
- [Apache Kafka KRaft](https://kafka.apache.org/documentation/#kraft)
- [Strimzi KRaft Mode](https://strimzi.io/docs/operators/latest/deploying.html#deploying-kafka-cluster-kraft)

## License

MIT

## Author

kadirsahan
