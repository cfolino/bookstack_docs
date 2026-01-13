This page documents the **monthly internal TLS certificate issuance and deployment schedule** across the homelab.
All jobs run on the **10th of each month**, staggered deliberately to avoid CA contention, overlapping restarts, or monitoring noise.

The schedule is executed via **Semaphore**, using Ansible playbooks that delegate signing operations to the internal CA.

---

## Execution Strategy

- **Single CA** → staggered signing operations
- **Fail-fast behavior** → downstream jobs assume upstream stability
- **Monitoring-first** → Grafana and Prometheus go first
- **Core infrastructure last** → OMV, PVE, Pi-hole

All times below are **local system time**.

---

## Certificate Rotation Schedule (Cron)

<details>
<summary><strong>02:00 – Monitoring & Automation Control Plane</strong></summary>

| Service     | Cron Expression | Time  |
|------------|-----------------|-------|
| Grafana    | `0  2 10 * *`   | 02:00 |
| BookStack  | `15 2 10 * *`   | 02:15 |
| Semaphore  | `30 2 10 * *`   | 02:30 |
| Prometheus | `45 2 10 * *`   | 02:45 |

**Notes**
- Grafana first to preserve dashboards during the cycle
- Semaphore rotated early to avoid cert expiry during later jobs
- Prometheus last to avoid scrape errors during restarts

</details>

---

<details>
<summary><strong>03:00 – Workloads & Services</strong></summary>

| Service        | Cron Expression | Time  |
|---------------|-----------------|-------|
| Node Exporter | `0  3 10 * *`   | 03:00 |
| Home VM       | `15 3 10 * *`   | 03:15 |
| PBS           | `30 3 10 * *`   | 03:30 |
| NPM           | `45 3 10 * *`   | 03:45 |

**Notes**
- Node Exporter rotated before dependent dashboards
- PBS isolated to prevent backup disruption
- Nginx Proxy Manager last due to broad service fan-out

</details>

---

<details>
<summary><strong>04:00 – Core Infrastructure</strong></summary>

| Service | Cron Expression | Time  |
|--------|-----------------|-------|
| OMV    | `0  4 10 * *`   | 04:00 |
| PVE    | `15 4 10 * *`   | 04:15 |
| Pi-hole| `30 4 10 * *`   | 04:30 |

**Notes**
- OMV first to ensure storage availability
- PVE rotated after storage is confirmed healthy
- Pi-hole last to minimize DNS disruption window

</details>

---

## Validation Checklist

After the schedule completes:

- [ ] All hosts present valid certs signed by internal CA
- [ ] No Grafana TLS or datasource errors
- [ ] Prometheus targets remain `UP`
- [ ] No Semaphore job failures
- [ ] No Pi-hole DNS interruption alerts
- [ ] Blackbox TLS expiry panels updated

---

## Change Control

- Schedule runs **monthly on the 10th**
- Changes require updating:
  - Semaphore cron
  - BookStack documentation
  - Any dependent monitoring assumptions

---
