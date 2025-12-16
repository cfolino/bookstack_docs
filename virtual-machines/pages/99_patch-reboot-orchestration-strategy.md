---

This page defines the **global patching and reboot orchestration model** for the homelab.
It explains **why different systems use different automation paths**, how dependencies are respected, and how Ansible enforces **safe execution order** without implicit assumptions.

This is the authoritative reference for understanding **how all patching workflows fit together**.

---

## Core Design Philosophy

The environment follows four non-negotiable principles:

1. **Authority must be respected**
2. **One reboot per system per cycle**
3. **The hypervisor owns VM lifecycle**
4. **Infrastructure before services**

Every playbook and role exists to uphold these rules.

---

## System Classification

All systems fall into one of four orchestration classes:

| Class | Examples | Reboot Authority |
|-----|--------|----------------|
| Hypervisor | Proxmox VE | Self |
| Virtual Machines | PBS, Grafana, CA, Pi-hole | Proxmox |
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

Key characteristics:
- ZFS health verification
- Root filesystem space validation
- Optional dry-run mode
- Fire-and-forget reboot
- Control node downtime is expected

> The hypervisor is patched **alone** and **first**.

---

## Virtual Machines (VMs)

**Playbook:** `patch_all.yml`
**Authority:** Proxmox VE
**Reboot Method:** `qm stop / qm start`

Key characteristics:
- Guest patching only
- No in-guest reboots
- Mandatory QEMU Guest Agent
- Hypervisor-controlled lifecycle
- SSH + QEMU socket validation

Each VM is isolated:
- One VM failure does not halt others
- No cascading retries
- Deterministic shutdown behavior

---

## Proxmox Backup Server (PBS)

**Playbook:** `patch_pbs.yml`
**Authority:** Proxmox VE
**Reboot Method:** `qm reboot`

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
- Optional Btrfs health checks
- ICMP reachability validation
- Post-reboot service confirmation

OMV is treated as **stateful storage**, not ephemeral compute.

---

## Certificate Authority & NPM

**Playbook:** `issue_and_deploy_cert.yml`
**Authority:** Ansible control node
**Reboot Method:** Docker container restart only

Characteristics:
- No host reboot
- CSR generation on CA
- Signing without sudo
- Certificate deployment to NPM
- Targeted container restart

This workflow is **surgical and isolated** by design.

---

## Monitoring Infrastructure

**Playbook:** `install_monitoring_exporters.yml`

Handled separately because:
- No reboot required
- Binary installs only
- Service enablement
- Port exposure validation

Monitoring is deployed **after** patch stability is confirmed.

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
- Explicit validation of uptime

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
- No guest-initiated VM shutdowns
- No Proxmox API dependencies
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
- Explicit control over automation magic

It is designed to scale **without surprises**.

---

## Related Pages

- Proxmox VM Lifecycle & Patch Automation
- Proxmox VE Host Patching
- OpenMediaVault Automation
- Certificate Authority Automation
- Monitoring & Exporters

---
