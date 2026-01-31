# Tasmota Power Monitoring

Monitor your home power consumption using Tasmota-flashed smart plugs with Prometheus and Grafana.

## Overview

This project provides two deployment methods for monitoring power consumption from Tasmota smart plugs:

- **Real-time power monitoring** from multiple Tasmota devices
- **Historical data storage** with Prometheus time-series database
- **Rich visualizations** through Grafana dashboards
- **Host and container metrics** for comprehensive monitoring

## Deployment Options

Choose the deployment method that best fits your infrastructure:

### 🐳 [Docker Compose](docker/)
**Quick and simple deployment**
- Perfect for single-host setups
- Minimal configuration required
- Great for getting started or small deployments
- Uses local Docker volumes for persistence

[📖 Docker Setup Guide →](docker/)

### ☸️ [Kubernetes](kubernetes/)
**Scalable and production-ready**
- Designed for K3s/K8s clusters
- Uses Longhorn for persistent storage (adapt to your storage deployment)
- Highly available and scalable
- Includes advanced monitoring features

[📖 Kubernetes Setup Guide →](kubernetes/)

## Hardware Requirements

### Supported Smart Plugs
- **Kauf PLF12** (recommended) - Pre-flashed with Tasmota
- Any smart plug compatible with [Tasmota firmware](https://tasmota.github.io/docs/)

### Tasmota Firmware
If your plugs aren't pre-flashed:
- [Tasmota Installation Guide](https://tasmota.github.io/docs/Getting-Started/)
- [Latest Firmware Downloads](https://ota.tasmota.com/tasmota/release/)

## Available Metrics

Both deployment methods collect these power metrics:

| Metric | Description |
|--------|-------------|
| `tasmota_active_power_W` | Real-time power consumption (watts) |
| `tasmota_energy_kWh_total` | Cumulative energy usage |
| `tasmota_energy_today_kWh_total` | Today's energy consumption |
| `tasmota_voltage_V` | Line voltage |
| `tasmota_current_A` | Current draw |
| `tasmota_power_factor` | Power efficiency ratio |

## Quick Start

1. **Choose your deployment method** (Docker or Kubernetes)
2. **Configure device IP addresses** in the respective configuration files
3. **Deploy the stack** using the provided instructions
4. **Access Grafana** to view your power consumption dashboards

## Project Structure

```
tasmota-power-monitoring/
├── docker/                    # Docker Compose deployment
│   ├── README.md             # Docker setup guide
│   ├── docker-compose.monitoring.yml
│   ├── config/prometheus.yaml
│   └── *.json                # Grafana dashboards
├── kubernetes/               # Kubernetes deployment
│   ├── README.md            # K8s setup guide
│   └── manifests/           # K8s resource definitions
└── README.md                # This file
```

## Contributing

Contributions are welcome! Please feel free to submit issues, feature requests, or pull requests.

## License

This project is open source. Please check individual component licenses for specific terms.

## Acknowledgments

- [Tasmota Project](https://tasmota.github.io/) - Open source firmware for smart devices
- [Tasmota Power Exporter](https://github.com/astr0n8t/tasmota-power-exporter) - Prometheus exporter for Tasmota devices
- [Original Tutorial](https://ounapuu.ee/posts/2024/05/02/smartplugs/) - Inspiration for this project