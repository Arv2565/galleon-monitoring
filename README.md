# Galleon Monitoring

Multi-cluster Kubernetes add-on version monitoring system using Prometheus and Grafana. Deploy a single Helm chart on each galleon cluster to automatically collect component versions and display everything in a centralized Grafana dashboard.

## Architecture
```
Galleon Cluster 1                          Galleon Cluster 2
┌─────────────────────────┐                ┌─────────────────────────┐
│  Version Exporter Pod   │                │  Version Exporter Pod   │
│  (queries K8s API)      │                │  (queries K8s API)      │
│         ↓               │                │         ↓               │
│  Local Prometheus       │                │  Local Prometheus       │
│  (scrapes exporter)     │                │  (scrapes exporter)     │
│         │               │                │         │               │
└─────────┼───────────────┘                └─────────┼───────────────┘
          │                                          │
          │         remote-write                     │
          └──────────────┬───────────────────────────┘
                         ↓
              Central Monitoring Cluster
              ┌──────────────────────┐
              │  Central Prometheus  │
              │  (receives metrics)  │
              │         ↓            │
              │  Grafana Dashboard   │
              │  (single pane of     │
              │   glass for all      │
              │   galleons)          │
              └──────────────────────┘
```

The exporter gets its metrics into the **local** Prometheus via a ServiceMonitor. The existing remote-write configuration on each galleon cluster handles shipping metrics to the central Prometheus — the Helm chart does not need to configure this.

## Repository Structure
```
├── charts/
│   └── galleon-version-exporter/    # Helm chart deployed on each galleon
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── serviceaccount.yaml
│           ├── clusterrole.yaml
│           ├── clusterrolebinding.yaml
│           ├── configmap.yaml          # Python exporter script
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── servicemonitor.yaml
│           ├── remote-write-rbac.yaml  # RBAC for remote-write Job (optional)
│           └── remote-write-job.yaml   # Post-install Job (optional)
├── dashboards/
│   ├── galleon-versions.json           # Table view dashboard
│   └── galleon-fleet.json              # Fleet card view dashboard
└── README.md
```

## Components Monitored

The exporter tracks 8 add-on components in each galleon cluster:

| Component | Detection Method | Details |
|---|---|---|
| Kubernetes | API `/version` endpoint | Always present, gets `gitVersion` |
| EKS-A | Deployment image tag | Checks `eksa-controller-manager` in `eksa-system` namespace |
| Kubevirt | Custom Resource (CRD) | Reads `status.observedKubeVirtVersion` from KubeVirt CR |
| Rook-Ceph | Deployment image tag | Checks `rook-ceph-operator` in `rook-ceph` namespace |
| GPU Operator | Deployment image tag | Checks `gpu-operator` in `gpu-operator` namespace |
| CDI | Custom Resource (CRD) | Reads `status.observedVersion` from CDI CR |
| ArgoCD | Deployment image tag | Checks `argocd-server` in `argocd` namespace |
| Velero | Deployment image tag | Checks `velero` in `velero` namespace |

## Metrics Exposed

The exporter exposes two Prometheus metrics per component on port 9100:

| Metric | Labels | Description |
|---|---|---|
| `cluster_addon_info` | `cluster`, `component`, `version` | Version information. Value is always 1. |
| `cluster_addon_status` | `cluster`, `component`, `version` | Running status. `1` = running, `0` = not running, `-1` = not installed. |

Example output:
```
cluster_addon_info{cluster="my-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_status{cluster="my-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_info{cluster="my-galleon",component="argocd",version="N/A"} 1
cluster_addon_status{cluster="my-galleon",component="argocd",version="N/A"} -1
```

## Status Color Codes

| Color | Status Value | Meaning |
|---|---|---|
| 🟢 Green | `1` | Component is **running** — at least one pod has ready replicas |
| 🔴 Red | `0` | Component is **deployed but not running** — deployment exists but no ready replicas |
| ⚫ Grey | `-1` | Component is **not installed** — deployment or CRD not found on this cluster |

## Helm Chart Values

| Key | Default | Description |
|---|---|---|
| `galleonName` | `""` | **Required.** Human-readable name of the galleon. Used as the `cluster` label in all metrics. Cannot be changed via `helm upgrade` — requires uninstall and reinstall. |
| `image.repository` | `python` | Container image for the exporter. |
| `image.tag` | `3.12-slim` | Container image tag. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `scrapeInterval` | `300s` | How often Prometheus scrapes the exporter. |
| `serviceMonitorNamespace` | `monitoring` | Namespace where the ServiceMonitor is created. Must match where Prometheus looks for ServiceMonitors. |
| `serviceMonitorLabels` | `{}` | Extra labels for the ServiceMonitor (e.g. `release: kube-prometheus-stack`). Required when the local Prometheus has a `serviceMonitorSelector`. |
| `remoteWrite.enabled` | `false` | Enable automatic remote-write configuration via a post-install Job. Only needed if the cluster does not already have remote-write configured. |
| `remoteWrite.centralPrometheusUrl` | `""` | URL of the central Prometheus (e.g. `http://<CENTRAL_NODE_IP>:30090`). |
| `remoteWrite.prometheusName` | `""` | Name of the Prometheus CR on the galleon cluster. |
| `remoteWrite.prometheusNamespace` | `monitoring` | Namespace where the Prometheus CR lives. |

---

## Setup Guide

### Prerequisites

- Each galleon cluster must have `kube-prometheus-stack` (or equivalent Prometheus Operator) installed in the `monitoring` namespace.
- Each galleon cluster must have remote-write already configured on its local Prometheus, pointing to the central Prometheus.
- A central monitoring cluster must be available on the same network as all galleon clusters.
- `helm` and `kubectl` installed on your machine.

---

### Step 1: Set Up the Central Monitoring Cluster

Deploy `kube-prometheus-stack` on the central cluster with remote-write receiver enabled and Prometheus exposed via NodePort:
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

Wait for all pods to be ready:
```bash
kubectl get pods -n monitoring -w
```

Note the central Prometheus URL:
```bash
kubectl get nodes -o wide
```

The URL will be: `http://<NODE_IP>:30090`

Verify from a galleon cluster:
```bash
kubectl run test-net --rm -it --image=busybox --restart=Never \
  -- wget -qO- http://<CENTRAL_NODE_IP>:30090/api/v1/status/config 2>&1 | head -5
```

---

### Step 2: Deploy the Version Exporter on Each Galleon

Before installing, gather the required information from the galleon cluster:
```bash
export KUBECONFIG=~/galleon-kubeconfig.yaml

# Check if ServiceMonitor needs extra labels
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Then install the Helm chart:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

> **Note:** This assumes remote-write is already configured on the galleon's local Prometheus. The chart deploys the exporter and a ServiceMonitor in the `monitoring` namespace so the local Prometheus discovers and scrapes it. The existing remote-write pipeline then ships the metrics to the central Prometheus.

> **Important:** The `galleonName` should be a human-readable identifier for the galleon. This name appears in the Grafana dashboard. It cannot be changed via `helm upgrade` — you must uninstall and reinstall to change it.

#### Optional: Configure remote-write via the chart

If a galleon cluster does **not** already have remote-write configured, you can enable it in the Helm chart:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name> \
  --set remoteWrite.enabled=true \
  --set remoteWrite.centralPrometheusUrl=http://<CENTRAL_NODE_IP>:30090 \
  --set remoteWrite.prometheusName=<prometheus-cr-name>
```

#### Verify the installation
```bash
# Check exporter pod is running
kubectl get pods -n version-exporter

# Verify the ServiceMonitor was created in the monitoring namespace
kubectl get servicemonitor version-exporter-version-exporter -n monitoring
```

#### Verify data is flowing to the central Prometheus

Wait ~5 minutes, then check:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl port-forward svc/central-prom-kube-promethe-prometheus 9090:9090 -n monitoring &
sleep 5

curl -s "http://localhost:9090/api/v1/query?query=cluster_addon_info" | python3 -c "
import sys, json
data = json.loads(sys.stdin.read())
for r in data.get('data', {}).get('result', []):
    print(r['metric'].get('cluster'), r['metric'].get('component'), r['metric'].get('version'))
"
```

---

### Step 3: Set Up the Grafana Dashboard

Access the central Grafana:
```bash
export KUBECONFIG=~/central-cluster.yaml

# Get the Grafana admin password
kubectl get secret --namespace monitoring central-prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward Grafana
kubectl port-forward svc/central-prom-grafana 3060:80 -n monitoring --address 0.0.0.0 &
```

Open `http://localhost:3060` and log in with `admin` and the password.

#### Apply the dashboards
```bash
GRAFANA_PASS="<password-from-above>"

cd ~/galleon-monitoring

# Apply the table view dashboard
sed 's/DATASOURCE_UID/prometheus/g' dashboards/galleon-versions.json > /tmp/gv.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3060/api/dashboards/db \
  -d @/tmp/gv.json

# Apply the fleet card view dashboard
sed 's/DATASOURCE_UID/prometheus/g' dashboards/galleon-fleet.json > /tmp/gf.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3060/api/dashboards/db \
  -d @/tmp/gf.json
```

The dashboards will be available at:

- Table view: `http://localhost:3060/d/galleon-versions`
- Fleet view: `http://localhost:3060/d/galleon-fleet`

---

## Adding a New Galleon Cluster

Gather the required information:
```bash
export KUBECONFIG=~/new-galleon.yaml

kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Install with a single command:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

The new galleon will appear in the Grafana dashboard within 5 minutes.

---

## Renaming a Galleon

The `galleonName` cannot be changed via `helm upgrade` because Kubernetes deployment selector labels are immutable. To rename:
```bash
helm uninstall version-exporter -n version-exporter

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --set galleonName=<new-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

After renaming, restart the central Prometheus to flush stale metrics:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for fresh data to arrive.

---

## Adding a New Component to Monitor

Edit the exporter script in `charts/galleon-version-exporter/templates/configmap.yaml`:

1. Add to the default results dictionary:
```python
results = {
    ...
    "istio": {"version": "N/A", "status": -1},
}
```

1. Add detection logic. For deployment-based:
```python
ver, status = get_deployment_version("istio-system", "istiod")
if ver:
    results["istio"] = {"version": ver, "status": status}
```

For CRD-based:
```python
data = kube_get("/apis/<api-group>/<version>/<resource>")
if data and data.get("items"):
    ver = data["items"][0].get("status", {}).get("<version-field>", "")
    if ver:
        results["istio"] = {"version": ver, "status": check_namespace_running("istio-system")}
```

1. Add RBAC permissions if needed (in `templates/clusterrole.yaml`):
```yaml
- apiGroups: ["<api-group>"]
  resources: ["<resource-name>"]
  verbs: ["get", "list"]
```

1. Upgrade on all galleons:
```bash
helm upgrade version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --reuse-values
```

---

## Troubleshooting

### Exporter shows all N/A

Check the exporter logs:
```bash
kubectl logs -n version-exporter deployment/version-exporter-version-exporter
```

Verify RBAC permissions:
```bash
kubectl get clusterrole version-exporter-version-exporter -o yaml
```

### ServiceMonitor not picked up by Prometheus

Verify the ServiceMonitor exists in the `monitoring` namespace:
```bash
kubectl get servicemonitor version-exporter-version-exporter -n monitoring --show-labels
```

Check what label selector Prometheus expects:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Check what namespace selector Prometheus uses:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorNamespaceSelector}' ; echo
```

If the Prometheus is configured to only watch its own namespace, the ServiceMonitor in `monitoring` will be discovered. If it uses a different namespace selector, adjust the `serviceMonitorNamespace` value accordingly.

### Remote-write not working

If using the chart's built-in remote-write configuration, verify the config was applied:
```bash
kubectl get prometheus <PROMETHEUS_CR_NAME> -n monitoring -o jsonpath='{.spec.remoteWrite}' ; echo
```

If remote-write is managed outside the chart, verify it independently:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.remoteWrite}' ; echo
```

Verify network connectivity to the central Prometheus:
```bash
kubectl run test-net --rm -it --image=busybox --restart=Never \
  -- wget -qO- http://<CENTRAL_NODE_IP>:30090/api/v1/status/config
```

Check galleon Prometheus logs:
```bash
kubectl logs -n monitoring statefulset/<prometheus-statefulset> \
  -c prometheus --tail=20 | grep -i remote
```

### Dashboard shows no data

1. Set the time range to "Last 1 hour" in Grafana
2. Query the central Prometheus directly:
```bash
curl -s "http://<CENTRAL_NODE_IP>:30090/api/v1/query?query=cluster_addon_info"
```

### Stale data from old cluster names

Restart the central Prometheus:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for fresh data.

### Duplicate tiles in dashboard

Multiple exporters are running on the same cluster. Ensure only one exists:
```bash
kubectl get deployments -A | grep -i exporter
```

Remove duplicates. Only `version-exporter-version-exporter` in `version-exporter` namespace should exist.

---

## Complete Data Flow
```
helm install (on each galleon)
  │
  ├─► Version Exporter Pod (in version-exporter namespace)
  │     Queries Kubernetes API using in-cluster ServiceAccount
  │     Checks 8 components (Kubernetes, EKS-A, Kubevirt, Rook-Ceph,
  │       GPU Operator, CDI, ArgoCD, Velero)
  │     Caches results in background thread (every 5 minutes)
  │     Exposes metrics on port 9100 with cluster="<galleonName>" label
  │
  ├─► ServiceMonitor (in monitoring namespace, with configurable extra labels)
  │     Uses namespaceSelector to find the Service in the version-exporter namespace
  │     Tells local Prometheus: "scrape port 9100 on this service"
  │
  ▼
Local Prometheus (existing kube-prometheus-stack in each galleon)
  │     Discovers the ServiceMonitor in the monitoring namespace
  │     Scrapes the exporter every 5 minutes
  │     Remote-writes all metrics to the central Prometheus
  │     (remote-write is pre-configured on the cluster)
  │
  ▼
Central Prometheus (kube-prometheus-stack with enableRemoteWriteReceiver=true)
  │     Exposed via NodePort 30090
  │     Receives and stores metrics from all galleons
  │     All metrics have a "cluster" label identifying the source galleon
  │
  ▼
Central Grafana
       Single datasource pointing to the local central Prometheus
       Dashboard with cluster dropdown filter
       Table and fleet card views showing Component, Version, and Status
       New galleons auto-discovered via label_values() query
```
EOFcat > README.md << 'EOF'
# Galleon Monitoring

Multi-cluster Kubernetes add-on version monitoring system using Prometheus and Grafana. Deploy a single Helm chart on each galleon cluster to automatically collect component versions and display everything in a centralized Grafana dashboard.

## Architecture
```
Galleon Cluster 1                          Galleon Cluster 2
┌─────────────────────────┐                ┌─────────────────────────┐
│  Version Exporter Pod   │                │  Version Exporter Pod   │
│  (queries K8s API)      │                │  (queries K8s API)      │
│         ↓               │                │         ↓               │
│  Local Prometheus       │                │  Local Prometheus       │
│  (scrapes exporter)     │                │  (scrapes exporter)     │
│         │               │                │         │               │
└─────────┼───────────────┘                └─────────┼───────────────┘
          │                                          │
          │         remote-write                     │
          └──────────────┬───────────────────────────┘
                         ↓
              Central Monitoring Cluster
              ┌──────────────────────┐
              │  Central Prometheus  │
              │  (receives metrics)  │
              │         ↓            │
              │  Grafana Dashboard   │
              │  (single pane of     │
              │   glass for all      │
              │   galleons)          │
              └──────────────────────┘
```

The exporter gets its metrics into the **local** Prometheus via a ServiceMonitor. The existing remote-write configuration on each galleon cluster handles shipping metrics to the central Prometheus — the Helm chart does not need to configure this.

## Repository Structure
```
├── charts/
│   └── galleon-version-exporter/    # Helm chart deployed on each galleon
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── serviceaccount.yaml
│           ├── clusterrole.yaml
│           ├── clusterrolebinding.yaml
│           ├── configmap.yaml          # Python exporter script
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── servicemonitor.yaml
│           ├── remote-write-rbac.yaml  # RBAC for remote-write Job (optional)
│           └── remote-write-job.yaml   # Post-install Job (optional)
├── dashboards/
│   ├── galleon-versions.json           # Table view dashboard
│   └── galleon-fleet.json              # Fleet card view dashboard
└── README.md
```

## Components Monitored

The exporter tracks 8 add-on components in each galleon cluster:

| Component | Detection Method | Details |
|---|---|---|
| Kubernetes | API `/version` endpoint | Always present, gets `gitVersion` |
| EKS-A | Deployment image tag | Checks `eksa-controller-manager` in `eksa-system` namespace |
| Kubevirt | Custom Resource (CRD) | Reads `status.observedKubeVirtVersion` from KubeVirt CR |
| Rook-Ceph | Deployment image tag | Checks `rook-ceph-operator` in `rook-ceph` namespace |
| GPU Operator | Deployment image tag | Checks `gpu-operator` in `gpu-operator` namespace |
| CDI | Custom Resource (CRD) | Reads `status.observedVersion` from CDI CR |
| ArgoCD | Deployment image tag | Checks `argocd-server` in `argocd` namespace |
| Velero | Deployment image tag | Checks `velero` in `velero` namespace |

## Metrics Exposed

The exporter exposes two Prometheus metrics per component on port 9100:

| Metric | Labels | Description |
|---|---|---|
| `cluster_addon_info` | `cluster`, `component`, `version` | Version information. Value is always 1. |
| `cluster_addon_status` | `cluster`, `component`, `version` | Running status. `1` = running, `0` = not running, `-1` = not installed. |

Example output:
```
cluster_addon_info{cluster="my-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_status{cluster="my-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_info{cluster="my-galleon",component="argocd",version="N/A"} 1
cluster_addon_status{cluster="my-galleon",component="argocd",version="N/A"} -1
```

## Status Color Codes

| Color | Status Value | Meaning |
|---|---|---|
| 🟢 Green | `1` | Component is **running** — at least one pod has ready replicas |
| 🔴 Red | `0` | Component is **deployed but not running** — deployment exists but no ready replicas |
| ⚫ Grey | `-1` | Component is **not installed** — deployment or CRD not found on this cluster |

## Helm Chart Values

| Key | Default | Description |
|---|---|---|
| `galleonName` | `""` | **Required.** Human-readable name of the galleon. Used as the `cluster` label in all metrics. Cannot be changed via `helm upgrade` — requires uninstall and reinstall. |
| `image.repository` | `python` | Container image for the exporter. |
| `image.tag` | `3.12-slim` | Container image tag. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `scrapeInterval` | `300s` | How often Prometheus scrapes the exporter. |
| `serviceMonitorNamespace` | `monitoring` | Namespace where the ServiceMonitor is created. Must match where Prometheus looks for ServiceMonitors. |
| `serviceMonitorLabels` | `{}` | Extra labels for the ServiceMonitor (e.g. `release: kube-prometheus-stack`). Required when the local Prometheus has a `serviceMonitorSelector`. |
| `remoteWrite.enabled` | `false` | Enable automatic remote-write configuration via a post-install Job. Only needed if the cluster does not already have remote-write configured. |
| `remoteWrite.centralPrometheusUrl` | `""` | URL of the central Prometheus (e.g. `http://<CENTRAL_NODE_IP>:30090`). |
| `remoteWrite.prometheusName` | `""` | Name of the Prometheus CR on the galleon cluster. |
| `remoteWrite.prometheusNamespace` | `monitoring` | Namespace where the Prometheus CR lives. |

---

## Setup Guide

### Prerequisites

- Each galleon cluster must have `kube-prometheus-stack` (or equivalent Prometheus Operator) installed in the `monitoring` namespace.
- Each galleon cluster must have remote-write already configured on its local Prometheus, pointing to the central Prometheus.
- A central monitoring cluster must be available on the same network as all galleon clusters.
- `helm` and `kubectl` installed on your machine.

---

### Step 1: Set Up the Central Monitoring Cluster

Deploy `kube-prometheus-stack` on the central cluster with remote-write receiver enabled and Prometheus exposed via NodePort:
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

Wait for all pods to be ready:
```bash
kubectl get pods -n monitoring -w
```

Note the central Prometheus URL:
```bash
kubectl get nodes -o wide
```

The URL will be: `http://<NODE_IP>:30090`

Verify from a galleon cluster:
```bash
kubectl run test-net --rm -it --image=busybox --restart=Never \
  -- wget -qO- http://<CENTRAL_NODE_IP>:30090/api/v1/status/config 2>&1 | head -5
```

---

### Step 2: Deploy the Version Exporter on Each Galleon

Before installing, gather the required information from the galleon cluster:
```bash
export KUBECONFIG=~/galleon-kubeconfig.yaml

# Check if ServiceMonitor needs extra labels
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Then install the Helm chart:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

> **Note:** This assumes remote-write is already configured on the galleon's local Prometheus. The chart deploys the exporter and a ServiceMonitor in the `monitoring` namespace so the local Prometheus discovers and scrapes it. The existing remote-write pipeline then ships the metrics to the central Prometheus.

> **Important:** The `galleonName` should be a human-readable identifier for the galleon. This name appears in the Grafana dashboard. It cannot be changed via `helm upgrade` — you must uninstall and reinstall to change it.

#### Optional: Configure remote-write via the chart

If a galleon cluster does **not** already have remote-write configured, you can enable it in the Helm chart:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name> \
  --set remoteWrite.enabled=true \
  --set remoteWrite.centralPrometheusUrl=http://<CENTRAL_NODE_IP>:30090 \
  --set remoteWrite.prometheusName=<prometheus-cr-name>
```

#### Verify the installation
```bash
# Check exporter pod is running
kubectl get pods -n version-exporter

# Verify the ServiceMonitor was created in the monitoring namespace
kubectl get servicemonitor version-exporter-version-exporter -n monitoring
```

#### Verify data is flowing to the central Prometheus

Wait ~5 minutes, then check:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl port-forward svc/central-prom-kube-promethe-prometheus 9090:9090 -n monitoring &
sleep 5

curl -s "http://localhost:9090/api/v1/query?query=cluster_addon_info" | python3 -c "
import sys, json
data = json.loads(sys.stdin.read())
for r in data.get('data', {}).get('result', []):
    print(r['metric'].get('cluster'), r['metric'].get('component'), r['metric'].get('version'))
"
```

---

### Step 3: Set Up the Grafana Dashboard

Access the central Grafana:
```bash
export KUBECONFIG=~/central-cluster.yaml

# Get the Grafana admin password
kubectl get secret --namespace monitoring central-prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward Grafana
kubectl port-forward svc/central-prom-grafana 3060:80 -n monitoring --address 0.0.0.0 &
```

Open `http://localhost:3060` and log in with `admin` and the password.

#### Apply the dashboards
```bash
GRAFANA_PASS="<password-from-above>"

cd ~/galleon-monitoring

# Apply the table view dashboard
sed 's/DATASOURCE_UID/prometheus/g' dashboards/galleon-versions.json > /tmp/gv.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3060/api/dashboards/db \
  -d @/tmp/gv.json

# Apply the fleet card view dashboard
sed 's/DATASOURCE_UID/prometheus/g' dashboards/galleon-fleet.json > /tmp/gf.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3060/api/dashboards/db \
  -d @/tmp/gf.json
```

The dashboards will be available at:

- Table view: `http://localhost:3060/d/galleon-versions`
- Fleet view: `http://localhost:3060/d/galleon-fleet`

---

## Adding a New Galleon Cluster

Gather the required information:
```bash
export KUBECONFIG=~/new-galleon.yaml

kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Install with a single command:
```bash
helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<galleon-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

The new galleon will appear in the Grafana dashboard within 5 minutes.

---

## Renaming a Galleon

The `galleonName` cannot be changed via `helm upgrade` because Kubernetes deployment selector labels are immutable. To rename:
```bash
helm uninstall version-exporter -n version-exporter

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --set galleonName=<new-name> \
  --set serviceMonitorLabels.release=<prometheus-release-name>
```

After renaming, restart the central Prometheus to flush stale metrics:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for fresh data to arrive.

---

## Adding a New Component to Monitor

Edit the exporter script in `charts/galleon-version-exporter/templates/configmap.yaml`:

1. Add to the default results dictionary:
```python
results = {
    ...
    "istio": {"version": "N/A", "status": -1},
}
```

1. Add detection logic. For deployment-based:
```python
ver, status = get_deployment_version("istio-system", "istiod")
if ver:
    results["istio"] = {"version": ver, "status": status}
```

For CRD-based:
```python
data = kube_get("/apis/<api-group>/<version>/<resource>")
if data and data.get("items"):
    ver = data["items"][0].get("status", {}).get("<version-field>", "")
    if ver:
        results["istio"] = {"version": ver, "status": check_namespace_running("istio-system")}
```

1. Add RBAC permissions if needed (in `templates/clusterrole.yaml`):
```yaml
- apiGroups: ["<api-group>"]
  resources: ["<resource-name>"]
  verbs: ["get", "list"]
```

1. Upgrade on all galleons:
```bash
helm upgrade version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --reuse-values
```

---

## Troubleshooting

### Exporter shows all N/A

Check the exporter logs:
```bash
kubectl logs -n version-exporter deployment/version-exporter-version-exporter
```

Verify RBAC permissions:
```bash
kubectl get clusterrole version-exporter-version-exporter -o yaml
```

### ServiceMonitor not picked up by Prometheus

Verify the ServiceMonitor exists in the `monitoring` namespace:
```bash
kubectl get servicemonitor version-exporter-version-exporter -n monitoring --show-labels
```

Check what label selector Prometheus expects:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Check what namespace selector Prometheus uses:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorNamespaceSelector}' ; echo
```

If the Prometheus is configured to only watch its own namespace, the ServiceMonitor in `monitoring` will be discovered. If it uses a different namespace selector, adjust the `serviceMonitorNamespace` value accordingly.

### Remote-write not working

If using the chart's built-in remote-write configuration, verify the config was applied:
```bash
kubectl get prometheus <PROMETHEUS_CR_NAME> -n monitoring -o jsonpath='{.spec.remoteWrite}' ; echo
```

If remote-write is managed outside the chart, verify it independently:
```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.remoteWrite}' ; echo
```

Verify network connectivity to the central Prometheus:
```bash
kubectl run test-net --rm -it --image=busybox --restart=Never \
  -- wget -qO- http://<CENTRAL_NODE_IP>:30090/api/v1/status/config
```

Check galleon Prometheus logs:
```bash
kubectl logs -n monitoring statefulset/<prometheus-statefulset> \
  -c prometheus --tail=20 | grep -i remote
```

### Dashboard shows no data

1. Set the time range to "Last 1 hour" in Grafana
2. Query the central Prometheus directly:
```bash
curl -s "http://<CENTRAL_NODE_IP>:30090/api/v1/query?query=cluster_addon_info"
```

### Stale data from old cluster names

Restart the central Prometheus:
```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for fresh data.

### Duplicate tiles in dashboard

Multiple exporters are running on the same cluster. Ensure only one exists:
```bash
kubectl get deployments -A | grep -i exporter
```

Remove duplicates. Only `version-exporter-version-exporter` in `version-exporter` namespace should exist.

---

## Complete Data Flow
```
helm install (on each galleon)
  │
  ├─► Version Exporter Pod (in version-exporter namespace)
  │     Queries Kubernetes API using in-cluster ServiceAccount
  │     Checks 8 components (Kubernetes, EKS-A, Kubevirt, Rook-Ceph,
  │       GPU Operator, CDI, ArgoCD, Velero)
  │     Caches results in background thread (every 5 minutes)
  │     Exposes metrics on port 9100 with cluster="<galleonName>" label
  │
  ├─► ServiceMonitor (in monitoring namespace, with configurable extra labels)
  │     Uses namespaceSelector to find the Service in the version-exporter namespace
  │     Tells local Prometheus: "scrape port 9100 on this service"
  │
  ▼
Local Prometheus (existing kube-prometheus-stack in each galleon)
  │     Discovers the ServiceMonitor in the monitoring namespace
  │     Scrapes the exporter every 5 minutes
  │     Remote-writes all metrics to the central Prometheus
  │     (remote-write is pre-configured on the cluster)
  │
  ▼
Central Prometheus (kube-prometheus-stack with enableRemoteWriteReceiver=true)
  │     Exposed via NodePort 30090
  │     Receives and stores metrics from all galleons
  │     All metrics have a "cluster" label identifying the source galleon
  │
  ▼
Central Grafana
       Single datasource pointing to the local central Prometheus
       Dashboard with cluster dropdown filter
       Table and fleet card views showing Component, Version, and Status
       New galleons auto-discovered via label_values() query
```
