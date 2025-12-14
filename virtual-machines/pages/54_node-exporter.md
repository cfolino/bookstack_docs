## `node_exporter` — Host-Level Metrics Collector

### Purpose

Node Exporter exposes host-level system metrics (CPU, memory, disk, networking) to Prometheus. It is deployed on each VM or host you wish to monitor.

---

### Overview

* Runs on port `9100` by default.
* No config required—auto-discovers host metrics.
* Installed as a system service and enabled at boot.
* Metrics available at `http://grafanba.cfolino.com:9100/metrics`.

---

### Typical Setup Commands

```bash
# Download and install latest release (on target host)
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-*.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
sudo cp node_exporter-*/node_exporter /usr/local/bin/

# Create systemd unit
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF

# Enable and start
sudo systemctl daemon-reexec
sudo systemctl enable --now node_exporter
```

---

### Usage Notes

* Metrics exposed at: `http://<host>:9100/metrics`
* Ensure firewall allows inbound port 9100 from Prometheus server.
* Lightweight—safe to run on all VMs and bare metal nodes.

---

## Workflow Summary

| Step                           | Description                                                |
| ------------------------------ | ---------------------------------------------------------- |
| 1. Install Node Exporter       | Installed and enabled as service on each host              |
| 2. Prometheus scrapes targets  | Targets listed in `prometheus.yml`, pulled every 15s       |
| 3. Grafana displays dashboards | Prometheus added as a data source; dashboards use its data |

---

Let me know if you want sections added for:

* `alertmanager` (next step in stack)
* `pushgateway` or application-specific exporters
* auto-deploying Node Exporter via Ansible playbook or cloud-init
