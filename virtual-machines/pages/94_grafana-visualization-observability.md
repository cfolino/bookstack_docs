---

This page documents the Grafana deployment used as the visualization and observability layer for the homelab monitoring stack. Grafana consumes metrics exclusively from Prometheus and presents them through curated dashboards designed for infrastructure, VM, and service health monitoring.

Grafana is treated as a **read-only visualization layer** — it does not collect metrics directly.

---

## Role in the Monitoring Stack

Grafana is responsible for:
- Rendering dashboards from Prometheus metrics
- Providing a single-pane-of-glass view of infrastructure health
- Visualizing availability, performance, and trends
- Serving as the foundation for future alerting workflows

Grafana **does not scrape targets**. All metrics originate from:
- Node Exporter
- Blackbox Exporter
- Prometheus

---

## Host & Deployment Details

- Hostname: `grafana`
- OS: Ubuntu Server
- Deployment type: VM (Proxmox)
- Monitoring components on this host:
  - Grafana
  - Prometheus
  - Blackbox Exporter

---

## Data Sources

### Prometheus (Primary & Only Data Source)

- Type: Prometheus
- URL:
  ```text
  http://localhost:9090
  ```
- Access mode: Server
- Authentication: None (internal-only)
- TLS: Not used (trusted LAN)

All dashboards are built against this single Prometheus data source.

---

## Dashboard Strategy

Dashboards are intentionally structured to avoid clutter and redundancy.

### Design Principles
- Prefer **hostname-based labels**, not IPs
- Use consistent `instance` naming across panels
- Avoid dashboard-specific logic that duplicates Prometheus rules
- Favor reusable variables for hosts and services

---

## Core Dashboards

### Infrastructure / VM Overview
Provides a high-level status view:
- Host up/down status
- CPU usage
- Memory usage
- Disk usage
- Uptime

This dashboard is used as the primary operational view.

---

### Per-Host Detail Dashboards
Drill-down dashboards per node:
- CPU load, steal time
- Memory pressure
- Filesystem utilization
- Network throughput
- Reboot tracking

Powered entirely by Node Exporter metrics.

---

### Availability & Reachability
Based on Blackbox Exporter:
- ICMP reachability (ping)
- HTTP availability
- DNS resolution status

Useful for detecting:
- Network outages
- Service crashes
- TLS or proxy failures

---

## Metrics Used (Examples)

### Node Exporter
```promql
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_filesystem_avail_bytes
node_boot_time_seconds
```

### Blackbox Exporter
```promql
probe_success
probe_duration_seconds
probe_icmp_duration_seconds
```

---

## Labeling & Instance Normalization

Prometheus relabeling ensures:
- `instance` labels match hostnames (not raw IPs)
- Clean grouping in Grafana variables
- Consistent legend formatting

Grafana dashboards rely on this normalization for clarity.

---

## Variables & Templating

Common dashboard variables:
- `$host`
- `$instance`
- `$job`

Used to:
- Filter hosts dynamically
- Reuse dashboards across systems
- Reduce duplication

---

## Verification & Health Checks

### Grafana Service Status
```bash
systemctl status grafana-server
```

### Prometheus Data Source Test
Grafana UI → Data Sources → Prometheus → **Test**

### Exporter Visibility
Grafana UI → Explore:
```promql
up
```

This confirms whether Prometheus is successfully scraping targets.

---

## Security Posture

- Internal-only access (LAN)
- No anonymous external exposure
- No direct internet-facing dashboards
- Authentication handled locally (future hardening planned)
- No secrets stored in dashboards

---

## Backup & Persistence

- Dashboards are treated as **configuration**
- Source-of-truth is Ansible + Prometheus data
- Grafana can be rebuilt from:
  - Data source config
  - Dashboard exports
  - Prometheus metrics

---

## Maintenance Tasks

### Restart Grafana
```bash
systemctl restart grafana-server
```

### Upgrade Grafana
Handled via system patching workflows.

### Add New Dashboard
1. Confirm metrics exist in Prometheus
2. Build dashboard using normalized labels
3. Save with consistent naming
4. Validate against multiple hosts

---

## Future Enhancements

- Alertmanager integration
- Notification channels (email / webhook)
- Role-based access control
- Dashboard provisioning via Ansible
- TLS for Grafana UI using internal CA

---

## Related Pages
- Monitoring Exporters
- Prometheus Scrape Configuration
- Ansible Monitoring Automation
- Infrastructure Patching & Rebooting

---
