# PRD: Tasmota Power tasmota Migration to K3s

## 1. Overview

This document outlines the plan to migrate the existing Docker Compose-based Tasmota power tasmota stack to a Kubernetes (K3s) cluster. The goal is to create a more robust and scalable tasmota solution using Kubernetes resources. The K3s homelab utilizes Longhorn for persistent storage, which will be used for Prometheus and Grafana data.

## 2. Core Components

The following components from the `docker-compose.tasmota.yml` will be migrated:

*   **Prometheus:** For time-series data storage and scraping metrics.
*   **Grafana:** For data visualization and dashboards.
*   **Tasmota Power Exporters (x4):** For scraping power metrics from Tasmota smart plugs.
*   **Node Exporter & cAdvisor:** For host-level metrics.

---

## 3. Migration To-Do List

### 3.1. Kubernetes Namespace

-   [ ] Create a dedicated namespace for the tasmota stack to ensure logical separation from other applications.
    ```bash
    kubectl create namespace tasmota
    ```

### 3.2. Persistent Storage (Longhorn)

-   [ ] **Prometheus PersistentVolumeClaim (PVC):** Define and create a PVC for Prometheus data. This will use the default Longhorn `StorageClass`.
    *   **Name:** `prometheus-data-pvc`
    *   **Size:** (User to determine, e.g., `10Gi`)
    *   **Access Mode:** `ReadWriteOnce`

-   [ ] **Grafana PersistentVolumeClaim (PVC):** Define and create a PVC for Grafana data, settings, and dashboards.
    *   **Name:** `grafana-data-pvc`
    *   **Size:** (User to determine, e.g., `2Gi`)
    *   **Access Mode:** `ReadWriteOnce`

### 3.3. Configuration

-   [ ] **Prometheus ConfigMap:**
    *   Create a `ConfigMap` named `prometheus-config` from the existing `./config/prometheus.yaml` file.
    *   **Crucially, update the `scrape_configs` section** within the ConfigMap to target the new Kubernetes service names for the Tasmota exporters (e.g., `tasmota-power-1.tasmota.svc.cluster.local:8000`).

### 3.4. Application Deployments

-   [ ] **Prometheus:**
    *   Create a `Deployment` for the `prom/prometheus:v2.37.9` image.
    *   Mount the `prometheus-config` ConfigMap to `/etc/prometheus/`.
    *   Mount the `prometheus-data-pvc` PVC to `/prometheus`.
    *   Create a `ClusterIP` Service to expose the Prometheus port `9090` internally.

-   [ ] **Grafana:**
    *   Create a `Deployment` for the `grafana/grafana-oss:latest` image.
    *   Mount the `grafana-data-pvc` PVC to `/var/lib/grafana`.
    *   Create a `NodePort` or `LoadBalancer` Service to expose the Grafana port `3000` for external access.
    *   (Optional) Create an `Ingress` resource for user-friendly access via a hostname.

-   [ ] **Tasmota Power Exporters:**
    *   For each of the four Tasmota devices, create a corresponding `Deployment` and `Service`.
        *   **Deployment 1 (`tasmota-power-1`):**
            *   Image: `ghcr.io/astr0n8t/tasmota-power-exporter:latest`
            *   Environment Variable: `DEVICE_IP=192.168.86.62`
        *   **Service 1 (`tasmota-power-1`):**
            *   Expose port `8000`.
        *   **Deployment 2 (`tasmota-power-2`):**
            *   Image: `ghcr.io/astr0n8t/tasmota-power-exporter:latest`
            *   Environment Variable: `DEVICE_IP=192.168.86.63`
        *   **Service 2 (`tasmota-power-2`):**
            *   Expose port `8000`.
        *   **Deployment 3 (`tasmota-power-3`):**
            *   Image: `ghcr.io/astr0n8t/tasmota-power-exporter:latest`
            *   Environment Variable: `DEVICE_IP=192.168.86.54`
        *   **Service 3 (`tasmota-power-3`):**
            *   Expose port `8000`.
        *   **Deployment 4 (`tasmota-power-4`):**
            *   Image: `ghcr.io/astr0n8t/tasmota-power-exporter:latest`
            *   Environment Variable: `DEVICE_IP=192.168.86.52`
        *   **Service 4 (`tasmota-power-4`):**
            *   Expose port `8000`.

### 3.5. Node & Cluster Metrics

-   [ ] **Node Exporter & cAdvisor:**
    *   K3s includes components that expose node and container metrics, which Prometheus can often scrape by default.
    *   **Action:** First, investigate if the default K3s metrics endpoints are sufficient.
    *   If not, deploy `prometheus-node-exporter` as a `DaemonSet` to ensure it runs on every node in the cluster, mirroring the original setup's intent. The `cAdvisor` functionality is already embedded within the `kubelet` on each K3s node.

---

## 4. Post-Deployment Steps

-   [ ] **Verify Pods:** Ensure all pods in the `tasmota` namespace are in the `Running` state.
-   [ ] **Verify Prometheus Targets:** Access the Prometheus UI (via port-forwarding or Ingress) and check the "Targets" page to confirm that all Tasmota exporters and node metrics endpoints are `UP`.
-   [ ] **Configure Grafana:**
    *   Access the Grafana UI via its Service/Ingress.
    *   Add Prometheus as a new data source, using its internal Kubernetes service URL (e.g., `http://prometheus.tasmota.svc.cluster.local:9090`).
    *   Import the dashboard using the content from `grafana.json`.
-   [ ] **Decommission Docker Compose:** Once the K3s stack is confirmed to be working correctly, tear down the old environment.
    ```bash
    docker-compose -f docker-compose.tasmota.yml down
    ```
