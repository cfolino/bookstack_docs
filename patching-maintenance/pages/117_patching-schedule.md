This page defines the **authoritative reboot and maintenance schedule** managed via **Ansible + Semaphore**.

All reboots are:
- Fully automated
- Pre-scheduled
- Segmented by failure domain
- Validated post-reboot where required

Timezone: **America/New_York**

---

## Maintenance Cadence Overview

| Layer | Cadence | Isolation |
|-----|--------|-----------|
| Application / Internal VMs | Monthly | VM-only |
| DNS (Pi-hole) | Monthly | Primary / Backup split |
| Network (UniFi) | Quarterly | Network-only |
| Infrastructure (Control / Storage / Hypervisor) | Quarterly | Infra-only |

---

## Monthly Maintenance (Every Month)

### Day 2 — Application / Internal VMs
**Window:** 02:00–03:00

| Time | System |
|-----|--------|
| 02:00 | BookStack |
| 02:15 | Grafana |
| 02:30 | Nginx Proxy Manager |
| 02:45 | Certificate Authority (CA) |

Notes:
- Staggered execution
- Allows reboot, verification, and notification per system
- No DNS or infrastructure impact

---

### Day 3 — DNS Primary
**Window:** 02:00

| Time | System |
|-----|--------|
| 02:00 | Pi-hole PRIMARY |

Notes:
- Backup Pi-hole remains online
- Semaphore DNS dependency preserved

---

### Day 4 — DNS Backup
**Window:** 02:30

| Time | System |
|-----|--------|
| 02:30 | Pi-hole BACKUP |

Notes:
- Primary already validated
- DNS redundancy preserved

---

## Quarterly Maintenance
**Months:** January / April / July / October

---

### Day 5 — Network Maintenance ONLY (UniFi)
**Window:** 02:00–03:00

| Time | Scope |
|-----|-------|
| 02:00 | UniFi Controller / UDM / Switches / APs |

Rules:
- Network-only changes
- No VM, DNS, or infrastructure work
- Manual verification if anomalies are observed

---

### Day 6 — Infrastructure Maintenance ONLY
**Window:** 02:00–04:30

This window is **strictly ordered** and **gated by validation**.

| Time | System |
|-----|--------|
| 02:00 | Ansible Control Node — Patch & Reboot |
| 02:30 | Ansible Post-Reboot Validation (Gate) |
| 03:00 | OpenMediaVault (OMV) |
| 03:30 | Proxmox Backup Server (PBS) |
| 04:00 | Proxmox VE (PVE) |

Rules:
- No infrastructure work proceeds without successful validation
- Storage precedes backup
- Hypervisor is always last

---

## Validation & Safety Gates

### Ansible Post-Reboot Validation

Purpose:
- Confirm control-plane health after reboot
- Validate SSH reachability, DNS resolution, inventory integrity, and notifications

Failure behavior:
- Infrastructure maintenance is considered unsafe
- Manual intervention required before next run

---

## Observational Tasks (Non-Disruptive)

### UniFi Uptime Check

| Cadence | Time |
|------|------|
| Weekly (Friday) | 16:00 |

Notes:
- Read-only checks
- Used to validate stability heading into weekends

---

## Change Control Notes

- All maintenance runs via **Semaphore schedules**
- Repository updates do **not** trigger reboots
- Emergency changes are documented separately
- Any schedule change requires this page to be updated

---

## Summary

This schedule provides:
- Predictable downtime
- Clear isolation boundaries
- Dependency-safe ordering
- Human recovery windows
- Long-term operational clarity

This page is the **single source of truth** for reboot and maintenance timing.
