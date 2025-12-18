## Purpose

This page documents how **Prometheus** is configured to scrape metrics from all monitoring exporters deployed in the homelab.

The configuration aligns exactly with the **Ansible-deployed exporters**:

- **Node Exporter** on all infrastructure and Ubuntu VMs
- **Blackbox Exporter** on the Grafana VM

Prometheus acts as the **single source of truth** for metrics collection. Grafana consumes data exclusively from Prometheus.

---

## Prometheus Role in the Stack

Prometheus is responsible for:

- Scraping metrics endpoints
- Storing time-series data
- Serving metrics to Grafana dashboards
- Providing the foundation for alerting (future)

Grafana does **not** scrape hosts directly — all metrics flow through Prometheus.

---

## Exporters and Endpoints

### Node Exporter

- Deployed via Ansible to:
  - Infrastructure hosts
  - Ubuntu VMs
- Default endpoint:
  ```text
  http://<host>:9100/metrics
  ```
- Provides:
  - CPU, memory, disk
  - Network statistics
  - Filesystem and kernel metrics

---

### Blackbox Exporter

- Deployed **only** on the Grafana VM
- Default endpoint:
  ```text
  http://grafana:9115
  ```
- Used for:
  - HTTP(S) endpoint checks
  - ICMP reachability
  - DNS resolution testing

---

## Prometheus Configuration Location

Prometheus runs on the Grafana VM.

Primary configuration file:

```text
/etc/prometheus/prometheus.yml
```

This file defines:

- Global scrape settings
- Static scrape targets
- Blackbox probe jobs

---

## Scrape Configuration Overview

### Global Settings

Typical global configuration:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
```

---

### Node Exporter Scrape Job

Scrapes all Node Exporter endpoints:

```yaml
scrape_configs:
  - job_name: "node_exporter"
    static_configs:
      - targets:
          - <internal-host>:9100
          - <internal-host>:9100
          - <internal-host>:9100
          - <internal-host>:9100
```

Targets are explicitly listed to maintain deterministic visibility.

---

### Blackbox Exporter Scrape Job

Blackbox probes are configured using relabeling:

```yaml
  - job_name: "blackbox_http"
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://internal.example
          - https://internal.example
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: <internal-host>:9115
```

This allows Prometheus to probe services **through** the Blackbox Exporter.

---

## Validation

### Verify Targets in Prometheus UI

Open:

```text
https://internal.example/targets
```

All targets should show:

- **State:** UP
- **Last Scrape:** recent
- **No scrape errors**

---

### Manual Endpoint Verification

```bash
curl http://<host>:9100/metrics | head
curl https://internal.example/probe?target=https://internal.example
```

---

## Failure Modes

| Issue | Cause | Resolution |
|-----|------|-----------|
| Target DOWN | Exporter not running | Restart exporter service |
| Connection refused | Firewall blocked | Allow Prometheus → exporter |
| Blackbox probe fails | TLS or DNS issue | Validate certificate and DNS |
| No metrics in Grafana | Prometheus scrape failing | Check Prometheus targets |

---

## Security Considerations

- Exporters bind to **internal interfaces only**
- Prometheus is not exposed publicly
- Metrics are read-only and non-interactive
- TLS termination is handled upstream by NPM when applicable

---

## Summary

Prometheus serves as the **central metrics aggregation layer**.

All exporters are deployed and managed via Ansible, while Prometheus performs deterministic scraping and exposes metrics to Grafana for visualization and observability.

This design ensures consistency, traceability, and operational clarity across the monitoring stack.
