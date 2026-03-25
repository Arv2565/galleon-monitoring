# Galleon Monitoring

Multi-cluster Kubernetes add-on version monitoring system using Prometheus and Grafana.

## Repository Structure
```
├── charts/
│   └── galleon-version-exporter/    # Helm chart deployed on each galleon
├── dashboards/
│   ├── galleon-versions.json        # Table view dashboard
│   └── galleon-fleet.json           # Fleet card view dashboard
└── docs/
    └── README.md                    # Detailed setup documentation
```

## Quick Start

### 1. Set up the central monitoring cluster
```bash
export KUBECONFIG=~/central-cluster.yaml

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install central-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true \
  --set 'prometheus.prometheusSpec.tsdb.outOfOrderTimeWindow=10m' \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30090
```

### 2. Add a galleon cluster (single command)
```bash
export KUBECONFIG=~/galleon-kubeconfig.yaml

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set remoteWrite.enabled=true \
  --set remoteWrite.centralPrometheusUrl=http://<CENTRAL_NODE_IP>:30090 \
  --set remoteWrite.prometheusName=<prometheus-cr-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

### 3. Set up dashboards

See [docs/README.md](docs/README.md) for dashboard setup instructions.
