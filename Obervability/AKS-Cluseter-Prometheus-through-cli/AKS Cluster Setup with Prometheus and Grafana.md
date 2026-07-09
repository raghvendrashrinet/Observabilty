
## 1. Create AKS Cluster

| Bash | PowerShell | CMD |
|------|------------|-----|
| RESOURCE_GROUP="myResourceGroup"<br>AKS_NAME="myAKSCluster"<br>LOCATION="eastus" | $RESOURCE_GROUP = "myResourceGroup"<br>$AKS_NAME = "myAKSCluster"<br>$LOCATION = "eastus" | set RESOURCE_GROUP=myResourceGroup<br>set AKS_NAME=myAKSCluster<br>set LOCATION=eastus |

```
# Create resource group
az group create --name myResourceGroup --location eastus       -- Bash
az group create --name $RESOURCE_GROUP --location $LOCATION    -- Powershell
az group create --name %RESOURCE_GROUP% --location %LOCATION%  -- CMD


# Create AKS cluster
az aks create --resource-group $RESOURCE_GROUP --name $AKS_NAME --node-count 1 --node-vm-size Standard_B2s --generate-ssh-keys


# Get cluster credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME
```

## 2. Install Prometheus and Grafana

```bash
# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
 Generates YAML and Applies it safely Idempotent command,Works if namespace exists or not

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
  ✅ This deploys Prometheus, Grafana, Alertmanager, Node Exporter, and kube-state-metrics in the monitoring namespace.

Pods created:   kubectl get pods -n monitoring

Example output:
   prometheus-kube-prometheus-prometheus
   prometheus-grafana
   prometheus-node-exporter
   prometheus-kube-state-metrics
Architecture Created
 Kubernetes Cluster
        │
        ▼
 Node Exporter (node metrics)
        │
        ▼
 Prometheus (scrapes metrics)
        │
        ▼
 Grafana (visual dashboards)


```
## 3. Access Grafana      

```bash
# Get admin password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
 username: admin
 password: <decoded above>
Powershell
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" |
% { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

# Prometheus UI, Port-forward to access Prometheus Dashboard
kubectl port-forward -n monitoring svc/prometheus-opertated  9090:9090

# Garafana UI, Port-forward to access Grafana Dashboard , To access from your system
kubectl port-forward -n monitoring svc/prometheus-grafana 8080:80

# Alert Manager UI
kubectl port-forward -n monitoring svc/alertmanager-operated  9093:9093

# To check metric fetching on the node exporter/kube-state-matrics
- curl <node-exporter-svc-ip>:9100/metrics
- curl <prometheus-kube-state-metrics-svc-ip>:8080/metrics


# For production deployment
   Internet
     │
     ▼
  Ingress Controller
     │
     ▼
  Grafana Service


1️⃣ Production AKS Observability Architecture

                         Internet
                           │
                           ▼
                 Azure Application Gateway
                           │
                           ▼
                    Ingress Controller
               (NGINX / AGIC / Istio Gateway)
                           │
        ┌──────────────────┼───────────────────┐
        ▼                  ▼                   ▼
   Grafana UI          Application         DevOps Tools
   (Dashboards)          APIs
        │
        ▼
 ┌──--─────────────────────────────────────────-┐
 │           Observability Stack                │
 │                                              │
 │  Metrics        Logs            Tracing      │
 │                                              │
 │ Prometheus      Loki            Tempo        │
 │                                              │
 │ Node Exporter   FluentBit       OpenTelemetry|
 │ kube-state      Container logs  Traces       |
 │ metrics         Pod logs        Service traces
 │                                              │
 └────--─────────────────────────────────────-──┘
        │
        ▼
   Kubernetes Cluster
   (Pods / Services / Nodes)


2️⃣ Components Used in Enterprise
Metrics  : Collected by Prometheus
Sources:
  Kubernetes nodes
  Pods
  containers
  applications

Exporters used:
  Node Exporter → node metrics
  kube-state-metrics → cluster state
  Prometheus Operator → manages Prometheus

In the kube-prometheus-stack setup:
 Host/Node metrics: Node Exporter – deployed as a DaemonSet, collects CPU, memory, disk, and network usage from each AKS node. 
 Kubernetes infrastructure (Pods, Nodes, Deployments, etc.):  kube-state-metrics – queries the Kubernetes API and exposes metrics about object states (e.g., pod status, resource requests). 
 Applications hosted on the cluster: Custom application exporters (e.g., Prometheus client libraries in your app) or sidecar exporters. Prometheus scrapes metrics from the /metrics endpoint exposed by your application if a ServiceMonitor is configured. 


Logs: Collected using:
 Fluent Bit
 Grafana Loki

Flow:
 Pods
  │
  ▼
FluentBit
  │
  ▼
Loki
  │
  ▼
Grafana Logs

Distributed Tracing

Used for microservices debugging.

Common tool:

Grafana Tempo

Instrumentation via:

OpenTelemetry
 Trace flow:
  Service A
    │
    ▼
  Service B
    │
    ▼
  Service C

Tempo shows latency across services.
