# Kafka Cluster (KRaft Mode) - Helm Chart

Apache Kafka cluster running in KRaft mode using Strimzi operator - deployed via ArgoCD or Helm.

## Overview

This Helm chart deploys a highly configurable Kafka cluster using Strimzi's KafkaNodePool and Kafka custom resources. The cluster runs in KRaft mode, eliminating the need for ZooKeeper.

## Architecture

**KRaft Mode with Node Pools:**
- **Controllers**: Manage cluster metadata using Raft consensus
- **Brokers**: Handle client requests and data replication
- **No ZooKeeper**: Uses Kafka's native KRaft protocol for consensus

## Features

- ✅ **Helm Templated**: Fully parameterized for multiple deployments
- ✅ **KRaft Mode**: No ZooKeeper dependency
- ✅ **Node Pools**: Separate controller and broker nodes
- ✅ **Multiple Environments**: Dev and prod value files included
- ✅ **Configurable Resources**: CPU, memory, storage customizable
- ✅ **Flexible Listeners**: Internal (plain, TLS) and external (NodePort)
- ✅ **Entity Operators**: Topic and User management
- ✅ **Persistent Storage**: Data persistence with PVCs
- ✅ **GitOps Ready**: Managed via ArgoCD

## Prerequisites

- Kubernetes cluster (v1.19+)
- Strimzi Kafka Operator installed (v0.48.0+)
- Default StorageClass or specify storageClass in values
- Helm 3.x (for manual installation)
- ArgoCD (for GitOps deployment)

## Repository Structure

```
kraft-kafka-server/
├── Chart.yaml              # Helm chart metadata
├── values.yaml             # Default values (production)
├── values-dev.yaml         # Development environment values
├── values-prod.yaml        # Production environment values
├── templates/
│   ├── namespace.yaml      # Namespace template
│   ├── kafka-nodepool-controllers.yaml
│   ├── kafka-nodepool-brokers.yaml
│   └── kafka.yaml          # Main Kafka CR
└── README.md               # This file
```

## Configuration

### Default Values (Production)

| Component | Replicas | Memory | CPU | Storage |
|-----------|----------|--------|-----|---------|
| Controllers | 3 | 512Mi-1Gi | 500m-1000m | 10Gi |
| Brokers | 3 | 1Gi-2Gi | 500m-1000m | 10Gi |

### Key Parameters

```yaml
# Cluster identity
nameOverride: "kraft-kafka-cluster"
namespace: "kraft-kafka"

# Kafka version
kafka.version: "4.0.0"
kafka.metadataVersion: "4.0-IV0"

# Controllers
kafka.controllers.replicas: 3
kafka.controllers.storage.size: "10Gi"
kafka.controllers.storage.class: ""  # Empty = default StorageClass

# Brokers
kafka.brokers.replicas: 3
kafka.brokers.storage.size: "10Gi"
kafka.brokers.storage.class: ""

# Configuration
kafka.config.numPartitions: 3
kafka.config.defaultReplicationFactor: 3
kafka.config.minInSyncReplicas: 2

# Listeners
kafka.listeners.plain.enabled: true
kafka.listeners.tls.enabled: true
kafka.listeners.external.enabled: true
```

## Installation

### Option 1: Using ArgoCD (Recommended)

This chart is designed to be deployed via ArgoCD:

```bash
# Deployed automatically via app-of-apps pattern
kubectl apply -f https://raw.githubusercontent.com/kadirsahan/argocd-apps/main/app-of-apps.yaml
```

The ArgoCD Application manifest uses default values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kraft-kafka-server
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/kadirsahan/kraft-kafka-server.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values.yaml  # Use values-dev.yaml or values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: kraft-kafka
```

### Option 2: Using Helm Directly

```bash
# Clone the repository
git clone https://github.com/kadirsahan/kraft-kafka-server.git
cd kraft-kafka-server

# Install with default values (production)
helm install kafka-prod . -n kraft-kafka --create-namespace

# Install with development values
helm install kafka-dev . -f values-dev.yaml -n kraft-kafka-dev --create-namespace

# Install with custom values
helm install my-kafka . -f my-values.yaml -n my-namespace --create-namespace
```

### Option 3: Manual kubectl

```bash
# Render templates and apply
helm template kafka-prod . | kubectl apply -f -

# Or with custom values
helm template kafka-dev . -f values-dev.yaml | kubectl apply -f -
```

## Environment-Specific Deployments

### Development Environment

Uses `values-dev.yaml`:
- 1 controller, 1 broker (minimal resources)
- Smaller storage (5Gi)
- Lower replication factor (1)
- TLS disabled

```bash
helm install kafka-dev . -f values-dev.yaml -n kraft-kafka-dev --create-namespace
```

### Production Environment

Uses `values-prod.yaml`:
- 3 controllers, 3 brokers (high availability)
- Larger storage (50Gi controllers, 100Gi brokers)
- Full replication (factor: 3, min ISR: 2)
- TLS enabled

```bash
helm install kafka-prod . -f values-prod.yaml -n kraft-kafka --create-namespace
```

## Deploying Multiple Kafka Clusters

Deploy multiple independent Kafka clusters with different configurations:

```bash
# Cluster 1: Development
helm install kafka1 . -f values-dev.yaml \
  --set nameOverride=kafka1-cluster \
  --set namespace=kafka1 \
  --create-namespace

# Cluster 2: Staging
helm install kafka2 . \
  --set nameOverride=kafka2-cluster \
  --set namespace=kafka2 \
  --set kafka.controllers.replicas=1 \
  --set kafka.brokers.replicas=2 \
  --create-namespace

# Cluster 3: Production
helm install kafka3 . -f values-prod.yaml \
  --set nameOverride=kafka3-cluster \
  --set namespace=kafka3 \
  --create-namespace
```

## Verify Installation

```bash
# Check Helm release
helm list -n kraft-kafka

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

## Customization Examples

### Change Storage Size

```bash
helm upgrade kafka-prod . \
  --set kafka.controllers.storage.size=20Gi \
  --set kafka.brokers.storage.size=50Gi \
  -n kraft-kafka
```

### Scale Brokers

```bash
helm upgrade kafka-prod . \
  --set kafka.brokers.replicas=5 \
  -n kraft-kafka
```

### Disable External Listener

```bash
helm upgrade kafka-prod . \
  --set kafka.listeners.external.enabled=false \
  -n kraft-kafka
```

### Use Custom StorageClass

```bash
helm upgrade kafka-prod . \
  --set kafka.controllers.storage.class=fast-ssd \
  --set kafka.brokers.storage.class=fast-ssd \
  -n kraft-kafka
```

## Upgrading

### Upgrade Helm Release

```bash
# Upgrade with new values
helm upgrade kafka-prod . -f values-prod.yaml -n kraft-kafka

# Upgrade specific values
helm upgrade kafka-prod . --set kafka.brokers.replicas=5 -n kraft-kafka
```

### Upgrade Kafka Version

```bash
helm upgrade kafka-prod . \
  --set kafka.version=4.1.0 \
  --set kafka.metadataVersion=4.1-IV0 \
  -n kraft-kafka
```

**Note**: Follow Strimzi upgrade guidelines for version compatibility.

## Monitoring

```bash
# Check Kafka cluster status
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
helm upgrade kafka-prod . --set kafka.controllers.storage.class=your-storage-class -n kraft-kafka
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
# Using Helm
helm uninstall kafka-prod -n kraft-kafka

# Delete namespace
kubectl delete namespace kraft-kafka
```

**Warning**: This will delete all Kafka data. Ensure you have backups if needed.

## Values File Reference

See [values.yaml](values.yaml) for all available configuration options.

Key sections:
- `nameOverride`: Cluster name
- `namespace`: Target namespace
- `kafka.version`: Kafka version
- `kafka.controllers.*`: Controller node pool configuration
- `kafka.brokers.*`: Broker node pool configuration
- `kafka.config.*`: Kafka configuration
- `kafka.listeners.*`: Listener configuration
- `kafka.entityOperator.*`: Entity operator configuration

## Related Repositories

- [strimzi-operator](https://github.com/kadirsahan/strimzi-operator) - Strimzi operator installation
- [argocd-gitops](https://github.com/kadirsahan/argocd-gitops) - ArgoCD installation
- [argocd-apps](https://github.com/kadirsahan/argocd-apps) - App-of-Apps GitOps repository
- [kafka-connect](https://github.com/kadirsahan/kafka-connect) - Kafka Connect deployment

## Documentation

- [Strimzi Documentation](https://strimzi.io/documentation/)
- [Apache Kafka KRaft](https://kafka.apache.org/documentation/#kraft)
- [Strimzi KRaft Mode](https://strimzi.io/docs/operators/latest/deploying.html#deploying-kafka-cluster-kraft)
- [Helm Documentation](https://helm.sh/docs/)

## License

MIT

## Author

kadirsahan
