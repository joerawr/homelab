# Docker Compose Deployment

Deploy Tasmota power monitoring using Docker Compose for quick and simple setup.

## Overview

This setup monitors power consumption from Tasmota-flashed smart plugs and visualizes the data through Grafana dashboards. It includes:

- **Prometheus** - Time-series database for metrics collection
- **Grafana** - Data visualization and dashboards
- **Tasmota Power Exporters** - Scrapers for Tasmota device metrics
- **Node Exporter & cAdvisor** - Host and container metrics

## Prerequisites

- Docker and Docker Compose installed
- Tasmota-flashed smart plugs on your network
- Basic knowledge of your smart plug IP addresses

## Hardware Setup

### Tasmota Smart Plugs

This project was designed for **Kauf PLF12** plugs with Tasmota firmware:
- Purchase: [Kauf PLF12](https://kaufha.com/plf12/)
- Firmware: [Tasmota OTA](https://ota.tasmota.com/tasmota/release/)

The configuration supports 4 devices but can be easily modified for more or fewer plugs.

## Quick Start

1. **Clone and navigate to the directory**
   ```bash
   git clone <your-repo>
   cd tasmota-power-monitoring/docker
   ```

2. **Configure your device IPs**

   Edit `docker-compose.monitoring.yml` and update the IP addresses for your Tasmota devices:
   ```yaml
   environment:
     - DEVICE_IP=192.168.86.62  # Replace with your device IP
   ```

3. **Start the stack**
   ```bash
   docker-compose -f docker-compose.monitoring.yml up -d
   ```

4. **Access the services**
   - **Grafana**: http://localhost:3000 (admin/admin)
   - **Prometheus**: http://localhost:9090
   - **Tasmota Exporters**: http://localhost:8001-8004

## Configuration

### Device IP Addresses

Update the following in `docker-compose.monitoring.yml`:

```yaml
tasmota-1:
  environment:
    - DEVICE_IP=192.168.86.62  # Your first device

tasmota-2:
  environment:
    - DEVICE_IP=192.168.86.63  # Your second device

# ... repeat for tasmota-3 and tasmota-4
```

### Adding/Removing Devices

To modify the number of monitored devices:

1. **Add devices**: Copy a `tasmota-X` service block in `docker-compose.monitoring.yml`
2. **Update Prometheus config**: Add corresponding scrape targets in `config/prometheus.yaml`
3. **Update ports**: Ensure each exporter uses a unique port (8001, 8002, etc.)

Example for adding a 5th device:

```yaml
# In docker-compose.monitoring.yml
tasmota-5:
  image: ghcr.io/astr0n8t/tasmota-power-exporter:latest
  container_name: tasmota-power-5
  restart: always
  ports:
    - 8005:8000
  environment:
    - DEVICE_IP=192.168.86.XX
```

```yaml
# In config/prometheus.yaml
- job_name: "tasmota-5"
  static_configs:
    - targets: ["tasmota-power-5:8000"]
```

## Grafana Dashboard

### Initial Setup

1. Access Grafana at http://localhost:3000
2. Login with `admin`/`admin` (change password when prompted)
3. Add Prometheus data source:
   - URL: `http://prometheus:9090`
   - Access: Server (default)

### Import Dashboard

Import the provided dashboard using one of these files:
- `grafana.json` - Main power monitoring dashboard
- `tasmota_grafana_fixed.json` - Alternative dashboard version

**Import steps:**
1. Go to Dashboards → Import
2. Copy the JSON content from the file
3. Paste and click "Load"
4. Select your Prometheus data source

## Available Metrics

The Tasmota exporters provide these metrics:

| Metric | Description |
|--------|-------------|
| `tasmota_active_power_W` | Active power consumption in watts |
| `tasmota_apparent_power_VA` | Apparent power in volt-amperes |
| `tasmota_current_A` | Current draw in amperes |
| `tasmota_energy_kWh_total` | Total energy consumption |
| `tasmota_energy_today_kWh_total` | Today's energy consumption |
| `tasmota_energy_yesterday_kWh_total` | Yesterday's energy consumption |
| `tasmota_power_factor` | Power factor ratio |
| `tasmota_reactive_power_VAr` | Reactive power in volt-amperes reactive |
| `tasmota_voltage_V` | Voltage in volts |

## File Structure

```
docker/
├── docker-compose.monitoring.yml  # Main Docker Compose file
├── config/
│   └── prometheus.yaml            # Prometheus scrape configuration
├── grafana.json                   # Grafana dashboard export
├── tasmota_grafana_fixed.json     # Alternative dashboard
├── tasmota_1.json                 # Individual device dashboard
├── tasmota_2.json                 # Individual device dashboard
├── tasmota_3.json                 # Individual device dashboard
├── tasmota_4.json                 # Individual device dashboard
└── notes.txt                      # Development notes and references
```

## Troubleshooting

### Device Not Showing Data

1. **Check device accessibility**:
   ```bash
   curl http://<device-ip>/cm?cmnd=Status%208
   ```

2. **Verify exporter logs**:
   ```bash
   docker logs tasmota-power-1
   ```

3. **Check Prometheus targets**:
   - Go to http://localhost:9090/targets
   - Ensure all tasmota jobs show "UP" status

### Port Conflicts

If you have port conflicts, modify the port mappings in `docker-compose.monitoring.yml`:

```yaml
ports:
  - "9091:9090"  # Change from 9090:9090 if needed
```

## References

- [Original Tutorial](https://ounapuu.ee/posts/2024/05/02/smartplugs/)
- [Tasmota Power Exporter](https://github.com/astr0n8t/tasmota-power-exporter)
- [Tasmota Firmware](https://ota.tasmota.com/tasmota/release/)
- [Kauf PLF12 Setup](https://kaufha.com/plf12/)

## Data Persistence

- **Prometheus data**: Stored in `./data` directory (bind mount)
- **Grafana data**: Stored in Docker volume `grafana-data`

To backup your data:
```bash
# Backup Prometheus data
tar -czf prometheus-backup.tar.gz data/

# Backup Grafana data
docker run --rm -v tasmota_grafana-data:/data -v $(pwd):/backup alpine tar czf /backup/grafana-backup.tar.gz -C /data .
```

## Stopping the Stack

```bash
docker-compose -f docker-compose.monitoring.yml down
```

To remove all data:
```bash
docker-compose -f docker-compose.monitoring.yml down -v
rm -rf data/
```