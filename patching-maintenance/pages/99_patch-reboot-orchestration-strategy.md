---
## Purpose

This page defines the **global patching and reboot orchestration model** for the homelab.

It explains **why different systems use different automation paths**, how dependencies are respected, and how Ansible enforces **safe execution order without implicit assumptions**.

This is the authoritative reference for understanding **how all patching workflows fit together**.

---

## Core Design Philosophy

The environment follows four non-negotiable principles:

1. **Authority must be respected**
2. **One reboot per system per cycle**
3. **Guest OS owns guest lifecycle**
4. **Infrastructure before services**

Every playbook and role exists to uphold these rules.

---

## System Classification

All systems fall into one of four orchestration classes:

| Class | Examples | Reboot Authority |
|-----|--------|----------------|
| Hypervisor | Proxmox VE | Self |
| Virtual Machines | PBS, Grafana, CA, Pi-hole | Guest OS (Proxmox recovery only) |
| Bare Metal Services | OpenMediaVault | Self |
| Network Devices | UniFi | API / SSH |

Each class has **distinct constraints** and **non-overlapping automation**.

---

## Global Execution Order

Patch operations **must** follow this sequence:

```
1. Proxmox VE (hypervisor)
2. Infrastructure VMs
3. Service VMs
4. Bare metal services
5. Network devices
```

This ordering prevents:
- VM outages during hypervisor upgrades
- Storage disruption during backups
- Network loss during critical maintenance

---

## Proxmox VE (Hypervisor)

**Playbook:** `pve_patch.yml`
**Authority:** Proxmox node itself
**Reboot Method:** Direct `/sbin/reboot`
**Cadence:** Quarterly

Key characteristics:
- ZFS health verification
- Root filesystem space validation
- Kernel-aware patching
- Fire-and-forget reboot
- Control node downtime is expected

> The hypervisor is patched **alone**, **first**, and **in isolation**.

Proxmox reboots are **planned maintenance events**, not part of monthly VM patching.

---

## Virtual Machines (VMs)

**Playbook:** `patch_all.yml`
**Authority:** Guest OS
**Reboot Method:** `ansible.builtin.reboot`
**Verification:** SSH reachability

Key characteristics:
- Guest-controlled reboot
- Mandatory monthly reboot
- No reliance on systemd readiness state
- QEMU Guest Agent used for communication only
- Proxmox used **only for recovery**

Important rules:
- VM uptime in Proxmox is **not** a reboot indicator
- Guest OS uptime is authoritative
- Hypervisor power control is a safety net, not the default

Each VM is isolated:
- One VM failure does not halt others
- No cascading retries
- No stranded powered-off VMs

---

## Proxmox Backup Server (PBS)

**Playbook:** `patch_pbs.yml`
**Authority:** Guest OS with Proxmox recovery
**Reboot Method:** `ansible.builtin.reboot`

Additional safeguards:
- Active backup task detection
- Root disk free-space enforcement
- APT/dpkg lock guards
- Service-level health verification
- Package version reporting

PBS is **never patched while backups are running**.

---

## OpenMediaVault (Bare Metal)

**Playbook:** `patch_omv.yml`
**Authority:** Host OS
**Reboot Method:** `ansible.builtin.reboot`

Unique considerations:
- Samba graceful shutdown
- Filesystem health checks
- ICMP and SSH reachability validation
- Post-reboot service confirmation

OMV is treated as **stateful storage**, not ephemeral compute.

---

## Certificate Authority & NPM

**Playbook:** `issue_and_deploy_cert.yml`
**Authority:** Ansible control node
**Reboot Method:** None (service-level only)

Characteristics:
- No host reboot
- CSR generation on CA
- Certificate signing without sudo
- Deployment to NPM
- Targeted Docker container restarts only

This workflow is **surgical and isolated** by design.

---

## Monitoring Infrastructure

**Playbook:** `install_monitoring_exporters.yml`

Handled separately because:
- No reboot required
- Binary installs only
- Service enablement
- Port exposure validation

Monitoring is deployed **after patch stability is confirmed**.

---

## Network Devices (UniFi)

**Playbooks:**
- `unifi_preflight_check.yml`
- `unifi_reboot_devices.yml`
- `unifi_reboot_udm.yml`

Characteristics:
- API-driven reboots
- Optional SSH fallback
- Per-device execution
- Explicit uptime validation

Network devices are **last** to avoid orphaning management access.

---

## Dependency Enforcement

Dependencies are enforced by:

- Inventory scoping
- Explicit `delegate_to`
- Required variables (`vmid`, `proxmox_host`)
- Service-specific health checks
- Manual sequencing between playbooks (by design)

No automation attempts to infer topology.

---

## Why No “One Button Patch Everything”

A single monolithic playbook was **intentionally rejected** because it would:

- Obscure failure domains
- Hide reboot authority boundaries
- Encourage unsafe concurrency
- Reduce observability
- Increase blast radius

Each playbook is small, explicit, and auditable.

---

## Safety Guarantees

This orchestration guarantees:

- No double reboots
- No guest-initiated VM power-offs
- No reliance on Proxmox uptime as a signal
- No silent failures
- No hidden side effects

---

## Operator Expectations

Before patching:
- Confirm backups completed
- Confirm monitoring visibility
- Confirm SSH access

After patching:
- Validate dashboards
- Validate services
- Review email summaries

---

## Summary

This orchestration model prioritizes:

- Predictability over speed
- Authority over convenience
- Explicit control over automation behavior

It is designed to scale **without surprises** and reflects real system ownership boundaries.

---

## Related Pages

- Proxmox VM Lifecycle & Patch Automation
- Proxmox VE Host Patching
- OpenMediaVault Automation
- Certificate Authority Automation
- Monitoring & Exporters

---
