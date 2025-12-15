## `prometheus` — Time-Series Data Collection for Infrastructure Monitoring

### Purpose

Prometheus collects and stores time-series metrics from various homelab components (e.g., VMs, services, and exporters) and serves as the primary data source for Grafana dashboards.

---

### Overview

* Scrapes metrics from targets such as Node Exporter, Pi-hole Exporter, Pushgateway, etc.
* Exposes its own metrics on port `9090`.
* Configuration is defined in `prometheus.yml`.
* Integrated with Grafana via HTTP data source (`http://localhost:9090` or `http://grafana.cfolino.com:9090`).

---

### Prometheus Configuration (Key Snippet)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - '192.168.10.12:9100'   # grafana VM
        - '192.168.10.128:9100'  # proxmox host
        - '192.168.10.129:9100'  # pbs
        - '192.168.15.225:9100'  # omv
        - '192.168.30.20:9100'   # ansible control node
```

---

### Prometheus Access

| Component   | Value                                |
| ----------- | ------------------------------------ |
| UI URL      | `http://prometheus.cfolino.com:9090` |
| Config file | `/etc/prometheus/prometheus.yml`     |
| Systemd svc | `prometheus.service`                 |
| Logs        | `journalctl -u prometheus`           |

---

### Usage Notes

* Confirm Prometheus is scraping expected targets:
  `Status > Targets` in the Prometheus UI.
* Can be extended with additional exporters (e.g., `blackbox`, `node_exporter`, `alertmanager`).
* TLS or reverse proxy optional for LAN; consider if exposed externally.

---
