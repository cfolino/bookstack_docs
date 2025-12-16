---
This page documents how Prometheus is configured to scrape metrics from all monitoring exporters deployed in the homelab. It serves as the authoritative reference for what Prometheus scrapes, why it scrapes it, and how targets are verified.

The configuration aligns exactly with the Ansible-deployed exporters:
- **Node Exporter** on all infrastructure and Ubuntu VMs
- **Blackbox Exporter** on the Grafana VM

---

## Prometheus Role in the Stack

Prometheus is responsible for:
- Scraping metrics endpoints
- Storing time-series data
- Serving metrics to Grafana dashboards
- Providing the foundation for alerting (future)

Grafana does **not** scrape hosts directly — all metrics flow through Prometheus.

---

## Exporters & Endpoints

### Node Exporter
- Installed on:
  - Infrastructure hosts
  - Ubuntu VMs
- Metrics endpoint:
  - `http://<host>:9100/metrics`
- Purpose:
  - CPU, memory, disk, filesystem, load, uptime, network stats

### Blackbox Exporter
- Installed on:
  - Grafana VM only
- Metrics endpoint:
  - `http://grafana:9115/metrics`
- Purpose:
  - Active probing (ICMP, HTTP, DNS)

---

## Scrape Jobs Overview

| Job Name | Target | Port | Purpose |
|--------|--------|------|--------|
| `node_exporter` | All hosts | 9100 | Host-level metrics |
| `blackbox_icmp` | All hosts | 9115 | Ping / reachability |
| `blackbox_http` | Selected services | 9115 | HTTP availability |
| `blackbox_dns` | Internal DNS | 9115 | DNS resolution checks |

---

## Node Exporter Scrape Job

```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - bookstack:9100
          - grafana:9100
          - npm:9100
          - pihole:9100
          - pihole-backup:9100
          - ca:9100
          - pve:9100
          - pbs:9100
          - omv:9100
```

### Notes
- All targets correspond to hosts where `node_exporter` is deployed via Ansible
- DNS resolution is handled by internal Pi-hole + Unbound
- Metrics are scraped over the trusted LAN only

---

## Blackbox Exporter — ICMP Probing

```yaml
  - job_name: "blackbox_icmp"
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
          - bookstack
          - grafana
          - npm
          - pihole
          - pihole-backup
          - ca
          - pve
          - pbs
          - omv
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: grafana:9115
```

### What This Does
- Uses the Blackbox Exporter on Grafana
- ICMP probes every host
- Allows visibility into:
  - Reachability
  - Packet loss
  - Latency

---

## Blackbox Exporter — HTTP Probing

```yaml
  - job_name: "blackbox_http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://internal.example
          - https://internal.example
          - https://internal.example
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: grafana:9115
```

### What This Checks
- TLS availability
- HTTP response codes
- End-to-end service health via Nginx Proxy Manager

---

## Blackbox Exporter — DNS Probing

```yaml
  - job_name: "blackbox_dns"
    metrics_path: /probe
    params:
      module: [dns]
    static_configs:
      - targets:
          - <internal-host>
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: grafana:9115
```

### Purpose
- Confirms internal DNS resolution
- Detects Pi-hole / Unbound failures early

---

## Target Labeling Strategy

- `instance` label is rewritten to match hostnames or FQDNs
- Avoids raw IP-based labels in Grafana
- Enables clean dashboards and alerts

---

## Verification & Troubleshooting

### Check Prometheus Targets
```bash
http://<prometheus-host>:9090/targets
```

### Verify Metrics Endpoints
```bash
curl http://host:9100/metrics
curl http://grafana:9115/metrics
```

### Common Failures
| Symptom | Likely Cause |
|------|------------|
| Target DOWN | Exporter not running |
| Connection refused | Firewall or service stopped |
| No metrics | Wrong port or DNS issue |

---

## Security Considerations

- Exporters bind to LAN interfaces only
- No authentication on metrics endpoints (trusted network)
- No TLS (internal-only design)
- No credentials stored in Prometheus config

---

## Maintenance

### Add a New Host
1. Add host to Ansible inventory
2. Run exporter install playbook
3. Add target to Prometheus scrape config
4. Reload Prometheus

### Reload Prometheus
```bash
systemctl reload prometheus
```

---

## Related Pages
- Monitoring Exporters
- Grafana Dashboards
- Ansible Monitoring Automation
- Internal DNS Architecture

---
