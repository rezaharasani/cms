# Monitoring a Kubernetes (K8s) cluster 

It is happened using Prometheus and Grafana is one of the most common and powerful setups for observability. Below is a step-by-step guide explaining how to deploy, configure, and visualize metrics for your cluster.


## 🧩 Overview
**Prometheus** → collects metrics (from K8s components, nodes, pods, etc.)  
**Grafana** → visualizes metrics with dashboards
**Kube State Metrics** + **Node Exporter** + **cAdvisor** → provide detailed metrics


## 🏗️ Step-by-Step Setup
### 1. Install Prometheus and Grafana (Recommended: via Helm)
Helm simplifies deployment using community-maintained charts.

Add Helm repositories
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Create a monitoring namespace
```
kubectl create namespace monitoring
```

#### Install Prometheus stack:
The kube-prometheus-stack chart installs:
  * Prometheus
  * Alertmanager
  * Grafana
  * Node Exporter
  * Kube State Metrics

```
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring
```


### 2. Verify the Installation
Check that everything is running:
```
kubectl get pods -n monitoring
```

You should see pods like:
  * prometheus-kube-prometheus-stack-prometheus-*
  * alertmanager-*
  * grafana-*
  * kube-state-metrics-*
  * node-exporter-*


### 3. Access Grafana Dashboard
Grafana is installed as part of the stack. Get Grafana 'admin' user password by running:
```
kubectl --namespace monitoring get secrets kube-prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Access Grafana local instance:
```
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus" -oname)
kubectl --namespace monitoring port-forward $POD_NAME 3000:80 &
```

Then visit:
👉 http://127.0.0.1:3000


### 4. Explore Prebuilt Dashboards
The Prometheus stack includes several default Grafana dashboards, such as:
  * Kubernetes / Compute Resources / Cluster
  * Kubernetes / Nodes
  * Kubernetes / Pods
  * Kubernetes / API Server
  * Prometheus / Overview

You can view these under **Dashboards → Manage in Grafana**.


### 5. Access Prometheus (Optional)
You can also access Prometheus directly:
```
kubectl --namespace monitoring port-forward service/kube-prometheus-kube-prome-prometheus 9090:9090 &
```

Visit:
👉 http://127.0.0.1:9090  
You can query metrics (e.g., container_cpu_usage_seconds_total, kube_pod_info).


### 6. Configure Alerts (Optional)
The stack comes with Alertmanager. You can customize alert rules in the Helm values file:
```YAML
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'email-alert'
    receivers:
      - name: 'email-alert'
        email_configs:
          - to: 'ops@example.com'
```

Then apply changes via:
```
helm upgrade kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```


### 7. (Optional) Expose Grafana/Prometheus via Ingress
If you use an ingress controller:
```YAML
grafana:
  ingress:
    enabled: true
    hosts:
      - grafana.example.com
```

Then redeploy with:
```
helm upgrade kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml
```


## 🧠 Bonus: Custom Application Metrics
If you have custom apps (e.g., FastAPI, Go, etc.), you can expose `/metrics` endpoints and let Prometheus scrape them.

Example (FastAPI with `prometheus-fastapi-instrumentator`):
```Python
from prometheus_fastapi_instrumentator import Instrumentator
app = FastAPI()
Instrumentator().instrument(app).expose(app)
```

Then configure Prometheus to scrape that endpoint via `ServiceMonitor` or `PodMonitor` (supported by the Helm chart).


## ✅ Summary

| Component              | Purpose                | Installed by          |
|------------------------|------------------------|-----------------------|
| **Prometheus**         | Collects metrics       | kube-prometheus-stack |
| **Grafana**            | Visualizes metrics     | kube-prometheus-stack |
| **Alertmanager**       | Handles alerts         | kube-prometheus-stack |
| **Node Exporter**      | Node-level metrics     | kube-prometheus-stack |
| **Kube State Metrics** | Cluster object metrics | kube-prometheus-stack |

