## Purpose

This document defines the **authoritative patching and reboot behavior** for the Proxmox VE (PVE) node in the cfolino.com infrastructure.

PVE is the **hypervisor and infrastructure control plane**. Any maintenance on this host has direct impact on:
- All hosted virtual machines
- The Ansible control node
- Backup orchestration
- Network-adjacent services

As such, PVE patching is intentionally **strict, isolated, and non-interactive**.

---

## Role & Playbook

- **Role:** `pve_patch`
- **Playbook:** `playbooks/pve_patch.yml`
- **Execution Scope:** Single host (`pve`)
- **Automation Authority:** Ansible via Semaphore

PVE patching is never run as part of a group operation.

---

## Preconditions (Hard Gates)

The following conditions **must be met** before patching proceeds. Failure of any check **aborts execution** before package changes occur.

### Proxmox Environment Validation
- `pveversion` must be present and executable
- Confirms host is a valid Proxmox VE system

### ZFS Pool Health
- Command: `zpool status -x`
- Pools must report healthy
- Any ZFS error state is treated as a stop condition

### Root Filesystem Free Space
- Command: `df -h /`
- If root filesystem usage exceeds ~90%, execution fails
- Prevents upgrades on critically constrained systems

### Dry-Run Awareness
- If `pve_patch_dry_run=true`, no upgrades or reboot occur
- Dry-run mode is explicitly announced in output

---

## Patch Behavior

### Package Management
- Apt cache is refreshed
- Available upgrades are **always displayed**, even in dry-run mode
- System upgraded using:
  - `apt dist-upgrade`
  - `autoremove` and `autoclean` enabled

### Visibility Guarantees
- Upgradable packages are listed prior to upgrade
- Package activity is captured for notification reporting

### Reboot Requirement Detection
- `/var/run/reboot-required` is checked
- Result is reported but **does not gate reboot**
- PVE reboots are unconditional when enabled

---

## Reboot Semantics (Critical)

### Reboot Trigger
- Reboot is executed via `/sbin/reboot`
- Reboot is **asynchronous** and fire-and-forget

### Expected Behavior
- SSH connectivity will drop immediately
- Ansible control node VM may go offline
- Semaphore job may briefly appear stalled
- These behaviors are **expected and correct**

### Error Handling
- SSH failure during reboot is ignored
- No attempt is made to wait for reconnection
- Reboot success is assumed once command is issued

This approach avoids false negatives and ensures reliable execution in hypervisor contexts.

---

## Notification Behavior

### Timing
- Notification is sent **before reboot is initiated**
- Guarantees receipt even when control-plane connectivity is lost

### Contents
Notifications include:
- Proxmox VE version
- ZFS pool health summary
- Root filesystem usage
- Cluster status (if applicable)
- List of upgradable packages
- Reboot-required indicator
- Explicit reboot intent

Notifications serve as the authoritative audit record for PVE maintenance.

---

## Scheduling Constraints (Non-Negotiable)

PVE patching **must follow these rules**:

- Never scheduled concurrently with any other infrastructure job
- Never scheduled during DNS maintenance windows
- Never scheduled during PBS maintenance
- Always scheduled in its own dedicated window
- Must complete before Ansible control node patching

PVE maintenance defines the outer boundary of infrastructure patching.

---

## Expected “Failure” Conditions (Not Errors)

The following conditions are **normal** and do not indicate failure:

- SSH disconnect during reboot
- Loss of Ansible control node connectivity
- Semaphore job not receiving post-reboot callbacks

These outcomes are a consequence of correct hypervisor reboot behavior.

---

## Post-Reboot Guarantees

After reboot:
- Proxmox VE completes a full system restart
- Hosted VMs resume according to Proxmox configuration
- No further automation actions are attempted
- Health verification occurs in subsequent jobs, not here

PVE does not self-validate post-reboot by design.

---

## Design Rationale Summary

PVE patching is intentionally:
- Conservative
- Isolated
- Non-interactive
- Explicitly disruptive

This ensures:
- No silent failures
- No partial upgrades
- No false automation success states

All downstream automation assumes PVE maintenance has already completed successfully.

---

## Change Control Notes

Any modification to this role must preserve:
- Pre-flight safety gates
- Asynchronous reboot behavior
- Notification-before-reboot ordering
- Single-host execution semantics

Violating these constraints increases the risk of infrastructure-wide impact.
