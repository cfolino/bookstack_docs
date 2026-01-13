---
## Purpose

This page documents the **bulk patching and reboot workflow for Ubuntu virtual machines**.

The workflow performs a **full OS upgrade followed by a mandatory guest-initiated reboot**, with **hypervisor-level recovery safeguards** applied only if the guest reboot fails.

The design prioritizes:

- Guest OS correctness
- SSH-verified readiness
- Failure-tolerant orchestration
- Zero stranded virtual machines

Hypervisor intervention is treated as a **recovery mechanism**, not the primary reboot authority.

---

## Scope

### Applies to

- All hosts in the `ubuntu_vms` inventory group
- Ubuntu-based application and service VMs

### Explicit exclusions

This workflow does **not** apply to:

- `ansible` (automation control plane)
- `omv` (stateful storage)
- `pbs` (backup infrastructure)
- `pve` (hypervisor)

These systems are patched using dedicated workflows.

---

## Entry Playbook

```text
playbooks/patch_all.yml
```

---

## Inventory Requirements

Each target host must define:

```text
inventory/host_vars/<host>.yml
```

Required variables:

- `vmid` — Proxmox VM ID
- `proxmox_host` — Proxmox node used for **recovery control only**

Failure to define these values will halt execution.

---

## Execution Flow (Authoritative)

### 1. QEMU Guest Agent Enforcement (`qemu_ga_fix`)

This role ensures **stable guest ↔ hypervisor communication**, not lifecycle control.

Actions:

- Install `qemu-guest-agent`
- Remove any GA-triggered shutdown hooks
- Reload systemd
- Ensure GA is enabled and running

**Important**

QEMU Guest Agent is **never allowed** to initiate guest shutdowns on service stop.

Proxmox already performs clean shutdowns natively when GA is present.

---

### 2. OS Patching + Mandatory Guest Reboot (`patch_reboot`)

This phase performs **patching and a required reboot on every run**:

- Update apt cache
- Apply `dist-upgrade`
- Autoremove and autoclean unused packages
- Reboot the guest OS unconditionally

Reboot verification criteria:

- SSH connectivity restored
- Successful execution of a basic command (`whoami`)
- **Systemd state is not used as a gating condition**

Guest OS uptime is the authoritative reboot indicator.

---

### 3. Hypervisor Recovery Restart (`proxmox_vm_restart`)

This role is **only executed if the guest reboot fails**.

It is never part of the normal execution path.

Recovery sequence:

1. Assert `vmid` and `proxmox_host` are defined
2. Attempt graceful shutdown:
   ```bash
   qm shutdown <vmid>
   ```
3. Force stop only if still running:
   ```bash
   qm stop <vmid>
   ```
4. Ensure VM is started (always-run safety net):
   ```bash
   qm start <vmid>
   ```
5. Wait for SSH connectivity
6. Wait for Ansible connection readiness

This guarantees that **no VM can remain powered off**, even if the play fails.

---

### 4. Post-Tasks

#### Pi-hole Health Checks

Executed only when the hostname contains `pihole`:

```yaml
when: "'pihole' in inventory_hostname"
```

Applies to both primary and backup Pi-hole instances.

---

#### Failure Notification

If patching or reboot fails:

- A recovery attempt is made via Proxmox
- A failure notification email is sent
- The play is marked as failed

Notification execution is delegated to the control node.

---

## Pre-Flight Validation

Before execution:

```bash
ansible-inventory -i inventory/hosts.yml --graph
ansible -i inventory/hosts.yml ubuntu_vms -m ping
```

Operational checks:

- Proxmox host reachable
- No overlapping maintenance windows
- VMIDs verified in inventory

---

## Execution Commands

Run normally:

```bash
ansible-playbook playbooks/patch_all.yml
```

Preview task flow:

```bash
ansible-playbook playbooks/patch_all.yml --list-tasks
```

Dry run (patch logic only; reboot still evaluated):

```bash
ansible-playbook playbooks/patch_all.yml --check
```

---

## Post-Patch Verification

Recommended guest-level checks:

```bash
ansible -i inventory/hosts.yml ubuntu_vms -m shell -a "uptime -p"
ansible -i inventory/hosts.yml ubuntu_vms -m shell -a "uname -r"
```

Successful execution should show:

- Recent uptime
- Updated kernel version
- SSH reachability across all hosts

---

## Safety Guarantees

This workflow guarantees:

- Only `ubuntu_vms` are targeted
- All targets reboot once per execution
- SSH is the sole readiness signal
- Proxmox is used only for recovery
- No VM can be stranded in a powered-off state

---

## Explicit Non-Goals

This workflow does **not**:

- Perform conditional reboots
- Rely on Proxmox VM uptime as a reboot indicator
- Patch infrastructure or storage nodes
- Perform application-level validation
- Coordinate external service outages

These concerns are handled by dedicated workflows.

---

## Summary

`patch_all.yml` provides a **predictable, failure-tolerant patch-and-reboot cycle** for Ubuntu virtual machines.

Reboots are **guest-initiated, SSH-verified, and hypervisor-safe**, aligning automation behavior with real system health rather than cosmetic hypervisor metrics.
