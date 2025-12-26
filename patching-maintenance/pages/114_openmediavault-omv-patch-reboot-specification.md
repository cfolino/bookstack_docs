## Purpose

This document defines the **authoritative patching and reboot behavior** for the OpenMediaVault (OMV) host in the cfolino.com infrastructure.

OMV functions as a **bare-metal storage appliance** providing network file services and backing critical data workflows. Unlike hypervisors or backup appliances, OMV directly interfaces with client systems and active file transfers.

As such, OMV patching prioritizes:
- Data integrity
- Graceful service handling
- Conditional disruption
- Clear auditability

---

## Role & Playbook

- **Role:** `omv_patch`
- **Playbook:** `playbooks/patch_omv.yml`
- **Execution Scope:** Single host (`omv`)
- **Automation Authority:** Ansible via Semaphore
- **Reboot Authority:** Local system reboot (conditional)

OMV patching is never executed concurrently with:
- PBS maintenance
- PVE maintenance
- Network control-plane changes

---

## Preconditions

OMV patching intentionally uses **lightweight prechecks**.

### System Identity Capture
- OS version and uptime are captured prior to patching
- Information is recorded in the patch summary for auditing

No aggressive gating is performed at this stage.
This is intentional to avoid unnecessary operational complexity on a storage appliance.

---

## Optional Filesystem Health Checks

### Btrfs Detection
- All mounted filesystems are inspected
- Any filesystem with `fstype=btrfs` is identified automatically

### Btrfs Health Collection
- `btrfs device stats` is executed per detected mount
- Failures do **not** abort patching
- Results are summarized for reporting only

### No Btrfs Present
- Explicitly recorded in the patch summary
- Treated as a valid and expected state

Filesystem checks are **observational**, not blocking.

---

## Patch Behavior

### Package Management
- Apt cache refreshed
- Full system upgrade performed using:
  - `apt dist-upgrade`
  - `autoremove` and `autoclean` enabled

### Reboot Requirement Detection
- `/var/run/reboot-required` is evaluated
- Patch task change state is tracked
- Reboot decision is derived from:
  - Kernel/system change **or**
  - Explicit reboot-required flag

OMV does **not** reboot unconditionally.

---

## Reboot Decision Logic (Critical)

Reboot occurs **only if required**.

```text
Reboot if:
- apt upgrade resulted in changes
OR
- reboot-required flag exists
