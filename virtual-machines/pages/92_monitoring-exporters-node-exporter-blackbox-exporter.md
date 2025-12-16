---
This section documents how Prometheus monitoring exporters are deployed and managed across the homelab using Ansible.

The exporters provide host-level metrics and active probing data to the Prometheus + Grafana monitoring stack. Deployment is automated, consistent, and idempotent across all applicable systems.

---

## Overview

Two exporters are deployed:

### Node Exporter
- Installed on:
  - All **infrastructure hosts**
  - All **Ubuntu VMs**
- Purpose:
  - Exposes host-level metrics (CPU, memory, disk, filesystem, load, uptime, etc.)
- Port:
  - `9100/tcp`
- Bind address:
  - `0.0.0.0:9100`

### Blackbox Exporter
- Installed only on:
  - **Grafana VM**
- Purpose:
  - Active probing (HTTP, ICMP, DNS)
  - Used for service availability and reachability monitoring
- Port:
  - `9115/tcp`
- Bind address:
  - `0.0.0.0:9115`

---

## Playbook Entry Point

### Playbook
```bash
ansible-playbook playbooks/install_monitoring_exporters.yml
```

### Playbook Structure
```yaml
- name: Install node_exporter on infrastructure and Ubuntu VMs
  hosts:
    - infrastructure
    - ubuntu_vms
  roles:
    - node_exporter

- name: Install blackbox_exporter on Grafana VM
  hosts: grafana
  roles:
    - blackbox_exporter
```

---

## Target Hosts

### Node Exporter Targets
The following inventory groups receive **node_exporter**:

- `infrastructure`
- `ubuntu_vms`

Current Ubuntu VM targets include:
- bookstack
- grafana
- npm
- pihole
- pihole-backup
- ca

### Blackbox Exporter Target
- grafana

---

## Node Exporter Role

### Defaults
```yaml
node_exporter_version: "1.7.0"
node_exporter_arch: "linux-amd64"
node_exporter_user: "node_exporter"
node_exporter_listen_address: "0.0.0.0:9100"
node_exporter_bin_dir: "/usr/local/bin"
node_exporter_service_name: "node_exporter"
```

### Installation Flow
1. Create dedicated system user (`node_exporter`, nologin)
2. Download official release archive from GitHub
3. Extract binary to `/usr/local/bin`
4. Set ownership to `node_exporter:node_exporter`
5. Install systemd service
6. Reload systemd
7. Enable and start service

### Systemd Unit
```ini
[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter --web.listen-address=0.0.0.0:9100
```

### Verification
```bash
systemctl status node_exporter
ss -tlnp | grep 9100
curl http://localhost:9100/metrics
```

---

## Blackbox Exporter Role

### Defaults
```yaml
blackbox_exporter_version: "0.25.0"
blackbox_exporter_arch: "linux-amd64"
blackbox_exporter_user: "blackbox_exporter"
blackbox_exporter_bin_dir: "/usr/local/bin"
blackbox_exporter_config_file: "/etc/blackbox_exporter/config.yml"
blackbox_exporter_listen_address: "0.0.0.0:9115"
```

### Installation Flow
1. Create dedicated system user (`blackbox_exporter`, nologin)
2. Create config directory (`/etc/blackbox_exporter`)
3. Download official release archive from GitHub
4. Extract binary to `/usr/local/bin`
5. Install static config file
6. Install systemd service
7. Reload systemd
8. Enable and start service

### Blackbox Modules Configured
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s

  dns:
    prober: dns
    timeout: 5s
    query_name: <internal-host>
    query_type: A
```

### Systemd Unit
```ini
ExecStart=/usr/local/bin/blackbox_exporter \
  --config.file=/etc/blackbox_exporter/config.yml \
  --web.listen-address=0.0.0.0:9115
```

### Verification
```bash
systemctl status blackbox_exporter
ss -tlnp | grep 9115
curl http://localhost:9115/metrics
```

---

## Prometheus Integration

- Prometheus scrapes:
  - Node Exporter via `:9100`
  - Blackbox Exporter via `:9115`
- Blackbox probes are defined in Prometheus scrape configuration
- Grafana dashboards consume Prometheus metrics directly

---

## Security Notes

- Exporters run as **dedicated, non-login system users**
- No shell access
- No sudo privileges
- Ports are exposed only within trusted LAN segments
- No TLS on exporter endpoints (internal-only, trusted network)

---

## Maintenance Notes

### Upgrade Exporters
- Bump version in role defaults
- Re-run playbook
- Services restart automatically

### Re-deploy / Repair
```bash
ansible-playbook playbooks/install_monitoring_exporters.yml
```

### Remove Exporters (Manual)
- Stop service
- Remove binary
- Remove systemd unit
- Remove system user

---

## Related Components
- Prometheus (scrape configuration)
- Grafana (dashboards)
- Ansible control node
- Infrastructure & VM patching workflows

---

**This page reflects the exact behavior of the Ansible playbook and roles as implemented.**
