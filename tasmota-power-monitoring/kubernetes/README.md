# Kubernetes Deployment

Deploy Tasmota power monitoring on Kubernetes/K3s for scalable, production-ready monitoring.

## Overview

This deployment method provides:

- **Self-contained stack** with Prometheus and Grafana
- **Persistent storage** using Longhorn (K3s default)
- **Scalable architecture** for multiple devices
- **Production-ready** configuration with proper resource management

## Prerequisites

- K3s/K8s cluster with Longhorn storage
- `kubectl` configured to access your cluster
- Basic knowledge of Kubernetes concepts

## Architecture

The deployment creates:

- **Namespace**: `monitoring` for logical isolation
- **Prometheus**: Time-series database with persistent storage
- **Grafana**: Visualization dashboard with persistent storage
- **Tasmota Exporters**: One deployment per device
- **Services**: Internal cluster communication

## Quick Start

1. **Clone the repository**
   ```bash
   git clone <your-repo>
   cd tasmota-power-monitoring/kubernetes
   ```

2. **Configure device IP addresses**

   Edit the deployment manifests to match your Tasmota device IPs:
   ```bash
   # Update IP addresses in manifests/tasmota-exporters.yaml
   # Change DEVICE_IP values to match your devices
   ```

3. **Deploy the stack**
   ```bash
   kubectl apply -f manifests/
   ```

4. **Verify deployment**
   ```bash
   kubectl get pods -n monitoring
   kubectl get pvc -n monitoring
   ```

5. **Access services**
   ```bash
   # Port-forward to access Grafana
   kubectl port-forward -n monitoring svc/grafana 3000:3000

   # Access at http://localhost:3000 (admin/admin)
   ```

## Configuration

### Device IP Addresses

Update the `DEVICE_IP` environment variables in `manifests/tasmota-exporters.yaml`:

```yaml
env:
- name: DEVICE_IP
  value: "192.168.86.62"  # Update with your device IP
```

### Storage Configuration

The deployment uses Longhorn with these default sizes:
- **Prometheus**: 10Gi
- **Grafana**: 2Gi

Modify in `manifests/storage.yaml` if needed:

```yaml
spec:
  resources:
    requests:
      storage: 10Gi  # Adjust as needed
```

### Scaling Devices

To monitor additional devices:

1. **Add new exporter deployment** in `manifests/tasmota-exporters.yaml`
2. **Update Prometheus config** in `manifests/prometheus-config.yaml`

Example for adding a 5th device:

```yaml
# In tasmota-exporters.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasmota-power-5
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tasmota-power-5
  template:
    metadata:
      labels:
        app: tasmota-power-5
    spec:
      containers:
      - name: exporter
        image: ghcr.io/astr0n8t/tasmota-power-exporter:latest
        env:
        - name: DEVICE_IP
          value: "192.168.86.XX"  # Your device IP
        ports:
        - containerPort: 8000
```

```yaml
# In prometheus-config.yaml (add to scrape_configs)
- job_name: "tasmota-5"
  static_configs:
    - targets: ["tasmota-power-5.monitoring.svc.cluster.local:8000"]
```

## Manifest Files

### Core Infrastructure
- `namespace.yaml` - Creates monitoring namespace
- `storage.yaml` - PersistentVolumeClaims for data storage

### Configuration
- `prometheus-config.yaml` - Prometheus scrape configuration
- `grafana-config.yaml` - Grafana dashboard imports (optional)

### Applications
- `prometheus.yaml` - Prometheus deployment and service
- `grafana.yaml` - Grafana deployment and service
- `tasmota-exporters.yaml` - Tasmota device exporters

## Accessing Services

### Internal Access (Pod-to-Pod)
- **Prometheus**: `http://prometheus.monitoring.svc.cluster.local:9090`
- **Grafana**: `http://grafana.monitoring.svc.cluster.local:3000`
- **Tasmota Exporters**: `http://tasmota-power-X.monitoring.svc.cluster.local:8000`

### External Access

#### Port Forward (Development)
```bash
# Grafana
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```

#### Ingress (Production)
Create an Ingress resource for external access:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
spec:
  rules:
  - host: grafana.yourdomai.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```

## Grafana Setup

1. **Access Grafana** (via port-forward or Ingress)
2. **Login** with `admin`/`admin`
3. **Add Prometheus data source**:
   - URL: `http://prometheus.monitoring.svc.cluster.local:9090`
   - Access: Server (default)
4. **Import dashboard**:
   - Copy JSON from `../docker/grafana.json`
   - Import via Dashboards → Import

## Monitoring and Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring
```

### View Logs
```bash
kubectl logs -n monitoring deployment/prometheus
kubectl logs -n monitoring deployment/grafana
kubectl logs -n monitoring deployment/tasmota-power-1
```

### Check Storage
```bash
kubectl get pvc -n monitoring
kubectl describe pvc prometheus-data -n monitoring
```

### Verify Prometheus Targets
1. Port-forward to Prometheus: `kubectl port-forward -n monitoring svc/prometheus 9090:9090`
2. Visit http://localhost:9090/targets
3. Ensure all tasmota jobs show "UP" status

## Backup and Restore

### Backup Prometheus Data
```bash
# Create backup of PVC
kubectl create job prometheus-backup --image=busybox -- tar czf /backup/prometheus-data.tar.gz -C /prometheus .
kubectl cp monitoring/prometheus-backup:/backup/prometheus-data.tar.gz ./prometheus-backup.tar.gz
```

### Backup Grafana Data
```bash
# Export dashboards via API
curl -u admin:admin http://localhost:3000/api/search | jq -r '.[] | .uid' | \
xargs -I {} curl -u admin:admin http://localhost:3000/api/dashboards/uid/{} > grafana-backup.json
```

## Uninstalling

```bash
# Remove all resources
kubectl delete namespace monitoring

# Note: PVCs may need manual cleanup depending on storage class settings
kubectl get pv | grep monitoring
```

## Integration with Existing Prometheus

If you have an existing Prometheus instance:

1. **Skip Prometheus deployment**: Remove `prometheus.yaml` from manifests
2. **Add scrape configs** to your existing Prometheus:
   ```yaml
   - job_name: "tasmota-devices"
     kubernetes_sd_configs:
     - role: service
       namespaces:
         names: ["monitoring"]
     relabel_configs:
     - source_labels: [__meta_kubernetes_service_label_app]
       regex: tasmota-power-.*
       action: keep
   ```
3. **Update Grafana data source** to point to your existing Prometheus

## Performance Considerations

- **Resource requests/limits**: Adjust based on your cluster capacity
- **Storage size**: Monitor usage and resize PVCs as needed
- **Scrape intervals**: Balance between data granularity and resource usage
- **Retention policies**: Configure Prometheus retention in deployment args

## Security Notes

- Change default Grafana admin password
- Consider network policies for additional security
- Use secrets for sensitive configuration data
- Regular security updates for container images