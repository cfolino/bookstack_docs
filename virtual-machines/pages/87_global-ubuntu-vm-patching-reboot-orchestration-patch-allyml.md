---

## Purpose

This page documents the **bulk patching and reboot workflow for Ubuntu virtual machines**.

This workflow performs a **full OS upgrade followed by a mandatory hypervisor-controlled restart** for all hosts in the `ubuntu_vms` inventory group.

The design prioritizes **deterministic reboots**, **clean shutdown semantics**, and **post-restart validation** over conditional or guest-driven reboot logic.

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
- `proxmox_host` — Proxmox node used for VM control

Failure to define these values will halt execution.

---

## Execution Flow (Authoritative)

### 1. QEMU Guest Agent Enforcement (`qemu_ga_fix`)

This role prepares the VM for **hypervisor-controlled restarts** by:

- Installing `qemu-guest-agent`
- Injecting a systemd shutdown hook:
  ```ini
  ExecStopPost=/usr/bin/qemu-ga --shutdown
  ```
- Reloading systemd
- Restarting the GA service
- Waiting for the GA virtio socket
- Verifying GA is running

This ensures **clean shutdown semantics** when the VM is stopped from Proxmox.

---

### 2. OS Patching (`patch_reboot`)

Despite the role name, this phase performs **patching only**:

- Updates apt cache
- Applies `dist-upgrade`
- Removes unused packages
- Ensures QEMU GA is running before restart

No reboot logic exists in this role.

---

### 3. Hypervisor-Controlled Restart (`proxmox_vm_restart`)

Every VM is restarted **unconditionally** using Proxmox commands:

Execution sequence:

1. Assert `vmid` is defined
2. Stop VM from Proxmox:
   ```bash
   qm stop <vmid> --skiplock
   ```
3. Pause for QEMU cleanup
4. Start VM:
   ```bash
   qm start <vmid>
   ```
5. Wait for SSH availability
6. Wait for GA virtio socket
7. Verify GA service is running

This guarantees:
- A single, clean reboot
- No reliance on guest OS reboot behavior
- Post-restart validation before play completion

---

### 4. Post-Tasks

#### Pi-hole Health Checks
Executed only when the hostname contains `pihole`:

```yaml
when: "'pihole' in inventory_hostname"
```

This applies to both primary and backup Pi-hole instances.

---

#### Notification
A completion notification is sent using the shared `notify_email` role.

Note:
- As written, notification inclusion may execute once per host unless constrained elsewhere.

---

## Pre-Flight Validation

Before execution:

```bash
ansible-inventory -i inventory/hosts.yml --graph
ansible -i inventory/hosts.yml ubuntu_vms -m ping
```

Operational checks:

- Proxmox host reachable
- No conflicting maintenance windows
- VMIDs verified in inventory

---

## Execution Commands

```bash
ansible-playbook playbooks/patch_all.yml
```

Preview execution:

```bash
ansible-playbook playbooks/patch_all.yml --list-tasks
```

Dry run:

```bash
ansible-playbook playbooks/patch_all.yml --check
```

---

## Post-Patch Verification

Recommended checks:

```bash
ansible -i inventory/hosts.yml ubuntu_vms -m shell -a "uptime -p"
ansible -i inventory/hosts.yml ubuntu_vms -m shell -a "uname -r"
```

All hosts should report recent uptime and updated kernels.

---

## Safety Guarantees

This workflow guarantees:

- Only `ubuntu_vms` are targeted
- All targets receive a clean reboot
- Reboot authority is centralized at the hypervisor
- QEMU GA readiness is enforced before and after restart

---

## Explicit Non-Goals

This workflow does **not**:

- Perform conditional reboots
- Patch infrastructure or storage nodes
- Pause external services or backups
- Perform application-level validation

Those concerns are handled by dedicated workflows.

---

## Summary

`patch_all.yml` provides a **deterministic, repeatable patch-and-restart cycle** for Ubuntu VMs.

Reboots are **mandatory, hypervisor-driven, and validated**, ensuring consistent post-maintenance state across the environment.

---
