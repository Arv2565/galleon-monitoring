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
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true

kubectl patch svc central-prom-kube-promethe-prometheus -n monitoring \
  --type merge -p '{"spec":{"type":"NodePort","ports":[{"port":9090,"targetPort":9090,"nodePort":30090,"name":"http-web"}]}}'
```

### 2. Add a galleon cluster
```bash
export KUBECONFIG=~/galleon-kubeconfig.yaml

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name>

# Check if ServiceMonitor needs a label
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo

# Configure remote-write
kubectl get prometheus -n monitoring
kubectl patch prometheus <PROMETHEUS_CR_NAME> -n monitoring \
  --type merge -p '{"spec":{"remoteWrite":[{"url":"http://<CENTRAL_NODE_IP>:30090/api/v1/write"}]}}'
```

### 3. Set up dashboards

See [docs/README.md](docs/README.md) for dashboard setup instructions.
