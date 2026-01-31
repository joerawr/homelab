# Restoring Grafana NodePort Access After Helm Upgrade

## Problem
After a Helm upgrade of the kube-prometheus-stack, Grafana service reverted from NodePort back to ClusterIP, losing external access on the previously configured port.

## Root Cause
Helm upgrades reset service configurations to chart defaults unless explicitly specified in values. The kube-prometheus-stack chart defaults Grafana service to ClusterIP type.

## Diagnosis Steps

### 1. Verify Current Service Configuration
```bash
kubectl get svc -A | grep grafana
kubectl get svc kps-grafana -n monitoring -o yaml
```

### 2. Check Helm Release History
```bash
helm list -n monitoring
helm history kps -n monitoring
helm get values kps -n monitoring
```

### 3. Identify Missing Configuration
Look for missing `grafana.service.type` in the Helm values output.

## Solution

### Restore NodePort Access
```bash
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring \
  --set grafana.service.type=NodePort \
  --reuse-values
```

### Verify the Fix
```bash
kubectl get svc kps-grafana -n monitoring
```

You should see output similar to:
```
NAME          TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kps-grafana   NodePort   10.43.184.136  <none>        80:XXXXX/TCP   Xd
```

## Prevention

### Option 1: Create a Values File
Create `monitoring-values.yaml`:
```yaml
grafana:
  service:
    type: NodePort
    nodePort: 31447  # Optional: specify exact port
  persistence:
    enabled: true
    size: 2Gi
    storageClassName: longhorn

prometheus:
  prometheusSpec:
    retention: 75d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
          storageClassName: longhorn
```

Then use:
```bash
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring -f monitoring-values.yaml
```

### Option 2: Always Include Service Type in Upgrades
```bash
helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring \
  --set grafana.service.type=NodePort \
  --reuse-values
```

## Key Takeaways
- Helm upgrades can reset configurations not explicitly defined in values
- Always review service configurations after Helm upgrades
- Use values files or explicit `--set` flags to persist custom configurations
- The `--reuse-values` flag preserves existing custom values during upgrades