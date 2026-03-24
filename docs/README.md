# Galleon Version Exporter

A Helm chart that deploys a Prometheus metrics exporter inside each galleon cluster to monitor the version and running status of Kubernetes add-on components. Designed to work with a centralized Grafana dashboard that provides a single pane of glass across all galleon clusters.

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

## Metrics Exposed (PromQL)

The exporter exposes two metrics per component on port 9100:

| Metric | Labels | Description |
|---|---|---|
| `cluster_addon_info` | `cluster`, `component`, `version` | Version information for each component. Value is always 1. |
| `cluster_addon_status` | `cluster`, `component`, `version` | Running status. `1` = running, `0` = deployed but not running, `-1` = not installed. |

Example output:

```
cluster_addon_info{cluster="houston-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_status{cluster="houston-galleon",component="kubevirt",version="v1.7.2"} 1
cluster_addon_info{cluster="houston-galleon",component="argocd",version="N/A"} 1
cluster_addon_status{cluster="houston-galleon",component="argocd",version="N/A"} -1
```

## Helm Chart Values

| Key | Default | Description |
|---|---|---|
| `galleonName` | `""` | **Required.** Human-readable name of the galleon cluster (e.g. `houston-galleon`, `alaska-edgelight`). Used as the `cluster` label in all metrics and displayed as the row header in the Grafana dashboard. Must be set at install time — changing it requires uninstall and reinstall. |
| `image.repository` | `python` | Container image for the exporter. |
| `image.tag` | `3.12-slim` | Container image tag. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `scrapeInterval` | `300s` | How often Prometheus scrapes the exporter. |

---

## Setup Guide

### Prerequisites

- Each galleon cluster must have `kube-prometheus-stack` (or equivalent Prometheus Operator setup) installed in the `monitoring` namespace.
- A central monitoring cluster must be available on the same network as all galleon clusters (for remote-write to work).
- `helm` and `kubectl` installed on your machine.

---

### Step 1: Set Up the Central Monitoring Cluster

Deploy `kube-prometheus-stack` on the central cluster with the remote-write receiver enabled:

```bash
export KUBECONFIG=~/central-cluster.yaml

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install central-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.enableRemoteWriteReceiver=true
```

Wait for all pods to be ready:

```bash
kubectl get pods -n monitoring -w
```

Expose Prometheus via NodePort so galleon clusters can reach it:

```bash
kubectl patch svc central-prom-kube-promethe-prometheus -n monitoring \
  --type merge -p '{"spec":{"type":"NodePort","ports":[{"port":9090,"targetPort":9090,"nodePort":30090,"name":"http-web"}]}}'
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

For each galleon cluster, install the Helm chart with a human-readable galleon name:

```bash
export KUBECONFIG=~/galleon-one.yaml

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=houston-galleon
```

> **Important:** The `galleonName` should be a human-readable identifier (e.g. `houston-galleon`, `alaska-edgelight`), not a technical cluster name. This name appears in the Grafana dashboard. It cannot be changed via `helm upgrade` — you must uninstall and reinstall to change it.

Repeat for each galleon:

```bash
export KUBECONFIG=~/galleon-two.yaml

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=alaska-edgelight
```

#### Verify the exporter is running

```bash
kubectl get pods -n version-exporter
```

#### Test the metrics output

```bash
kubectl port-forward svc/version-exporter-version-exporter 9100:9100 -n version-exporter &
sleep 5
curl localhost:9100/metrics
```

#### Handle ServiceMonitor label requirements

Check what label selector the local Prometheus expects:

```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

If it returns something like `{"matchLabels":{"release":"kube-prometheus-stack"}}`, patch the ServiceMonitor:

```bash
kubectl patch servicemonitor version-exporter-version-exporter -n version-exporter \
  --type merge -p '{"metadata":{"labels":{"release":"kube-prometheus-stack"}}}'
```

If it returns `{}`, no extra label is needed.

---

### Step 3: Configure Remote-Write from Each Galleon to the Central Prometheus

Find the Prometheus custom resource name on the galleon:

```bash
kubectl get prometheus -n monitoring
```

Patch it to enable remote-write:

```bash
kubectl patch prometheus <PROMETHEUS_CR_NAME> -n monitoring \
  --type merge \
  -p '{"spec":{"remoteWrite":[{"url":"http://<CENTRAL_NODE_IP>:30090/api/v1/write"}]}}'
```

Example:

```bash
kubectl patch prometheus kube-prom-stack-kube-prome-prometheus -n monitoring \
  --type merge \
  -p '{"spec":{"remoteWrite":[{"url":"http://10.20.22.215:30090/api/v1/write"}]}}'
```

#### Verify remote-write is working

Check the Prometheus logs:

```bash
kubectl logs -n monitoring statefulset/<prometheus-statefulset-name> \
  -c prometheus --tail=20 | grep -i remote
```

You should see `"Starting WAL watcher"` and `"Done replaying WAL"`.

Verify data arrived at the central Prometheus:

```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl port-forward svc/central-prom-kube-promethe-prometheus 9090:9090 -n monitoring &

curl -s "http://localhost:9090/api/v1/query?query=cluster_addon_info" | python3 -c "
import sys, json
data = json.loads(sys.stdin.read())
for r in data.get('data', {}).get('result', []):
    print(r['metric'].get('cluster'), r['metric'].get('component'), r['metric'].get('version'))
"
```

---

### Step 4: Set Up the Grafana Dashboard

Access the central Grafana:

```bash
export KUBECONFIG=~/central-cluster.yaml

# Get the Grafana admin password
kubectl get secret --namespace monitoring central-prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Port-forward Grafana
kubectl port-forward svc/central-prom-grafana 3000:80 -n monitoring --address 0.0.0.0 &
```

Open `http://localhost:3000` and log in with `admin` and the password.

#### Get the Prometheus datasource UID

```bash
GRAFANA_PASS="<password-from-above>"

curl -s -u "admin:${GRAFANA_PASS}" http://localhost:3000/api/datasources | python3 -c "
import sys, json
for ds in json.loads(sys.stdin.read()):
    if ds['type'] == 'prometheus':
        print(ds['uid'])
        break
"
```

#### Apply the dashboards

The dashboard JSON files are in the `dashboards/` directory. Before applying, replace `DATASOURCE_UID` in the JSON files with the actual UID, then push via the Grafana API:

```bash
# Apply the table view dashboard
sed "s/DATASOURCE_UID/<your-datasource-uid>/g" dashboards/galleon-versions.json > /tmp/gv.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3000/api/dashboards/db \
  -d @/tmp/gv.json

# Apply the fleet card view dashboard
sed "s/DATASOURCE_UID/<your-datasource-uid>/g" dashboards/galleon-fleet.json > /tmp/gf.json
curl -s -u "admin:${GRAFANA_PASS}" \
  -H "Content-Type: application/json" \
  -X POST http://localhost:3000/api/dashboards/db \
  -d @/tmp/gf.json
```

The dashboards will be available at:

- Table view: `http://localhost:3000/d/galleon-versions`
- Fleet view: `http://localhost:3000/d/galleon-fleet`

---

## Status Color Codes

| Color | Status Value | Meaning |
|---|---|---|
| 🟢 Green | `1` | Component is **running** — at least one pod has ready replicas |
| 🔴 Red | `0` | Component is **deployed but not running** — deployment exists but no ready replicas |
| ⚫ Grey | `-1` | Component is **not installed** — deployment or CRD not found on this cluster |

---

## Adding a New Galleon Cluster

1. Install the Helm chart:

```bash
export KUBECONFIG=~/new-galleon.yaml

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --create-namespace \
  --set galleonName=<new-galleon-name>
```

> **Important:** Use a human-readable name that identifies the physical galleon. This name cannot be changed without reinstalling.

1. Patch ServiceMonitor if needed:

```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo

# If labels are required:
kubectl patch servicemonitor version-exporter-version-exporter -n version-exporter \
  --type merge -p '{"metadata":{"labels":{"release":"<release-name>"}}}'
```

1. Configure remote-write:

```bash
kubectl get prometheus -n monitoring

kubectl patch prometheus <PROMETHEUS_CR_NAME> -n monitoring \
  --type merge \
  -p '{"spec":{"remoteWrite":[{"url":"http://<CENTRAL_NODE_IP>:30090/api/v1/write"}]}}'
```

The new galleon will appear in the Grafana dashboard within 5 minutes.

---

## Renaming a Galleon

The `galleonName` cannot be changed via `helm upgrade` because Kubernetes deployment selector labels are immutable. To rename:

```bash
helm uninstall version-exporter -n version-exporter

helm install version-exporter ./charts/galleon-version-exporter/ \
  --namespace version-exporter \
  --set galleonName=<new-name>
```

After renaming, restart the central Prometheus to flush stale metrics with the old name:

```bash
export KUBECONFIG=~/central-cluster.yaml
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for remote-write to push fresh metrics with the new name.

> **Note:** Don't forget to re-patch the ServiceMonitor label if the galleon's Prometheus requires it.

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

1. Add detection logic. For deployment-based detection:

```python
ver, status = get_deployment_version("istio-system", "istiod")
if ver:
    results["istio"] = {"version": ver, "status": status}
```

For CRD-based detection:

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

Verify the ServiceAccount has correct RBAC permissions:

```bash
kubectl get clusterrole version-exporter-version-exporter -o yaml
```

### ServiceMonitor not picked up by Prometheus

Check what label selector Prometheus expects:

```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}' ; echo
```

Ensure the ServiceMonitor has the matching label.

### Remote-write not working

Verify the central Prometheus has the receiver enabled:

```bash
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.enableRemoteWriteReceiver}' ; echo
```

Check the galleon Prometheus logs:

```bash
kubectl logs -n monitoring statefulset/<prometheus-statefulset> \
  -c prometheus --tail=20 | grep -i remote
```

Verify network connectivity:

```bash
kubectl run test-net --rm -it --image=busybox --restart=Never \
  -- wget -qO- http://<CENTRAL_NODE_IP>:30090/api/v1/status/config
```

### Dashboard shows no data

1. Check the time range in Grafana — set to "Last 5 minutes" or "Last 1 hour"
2. Query the central Prometheus directly to verify data exists:

```bash
curl -s "http://<CENTRAL_PROMETHEUS>:9090/api/v1/query?query=cluster_addon_info"
```

1. If data exists but dashboard is empty, verify the datasource UID in the dashboard JSON matches your Prometheus datasource

### Stale data from old cluster names

If you see old cluster names after renaming, restart the central Prometheus:

```bash
kubectl rollout restart statefulset prometheus-central-prom-kube-promethe-prometheus -n monitoring
```

Wait ~5 minutes for fresh data to arrive via remote-write.

---

## Complete Data Flow

```
Helm Chart (deployed inside each galleon)
  │
  ├─► Version Exporter Pod
  │     Queries Kubernetes API using in-cluster ServiceAccount
  │     Checks 8 components
  │     Caches results in background thread (every 5 minutes)
  │     Exposes metrics on port 9100
  │
  ├─► ServiceMonitor
  │     Tells local Prometheus: "scrape port 9100 on this service"
  │
  ▼
Local Prometheus (existing kube-prometheus-stack in each galleon)
  │     Scrapes the exporter every 5 minutes
  │     Stores metrics locally
  │
  │     remote-write configured via:
  │     kubectl patch prometheus ... --type merge
  │       -p '{"spec":{"remoteWrite":[{"url":"http://CENTRAL:30090/api/v1/write"}]}}'
  │
  ▼
Central Prometheus (kube-prometheus-stack with enableRemoteWriteReceiver=true)
  │     Receives and stores metrics from all galleons
  │     All metrics have a "cluster" label identifying the source galleon
  │
  ▼
Central Grafana
       Single datasource pointing to the local central Prometheus
       Dashboard with cluster dropdown filter
       Table and fleet card views showing Component, Version, and Status
```
