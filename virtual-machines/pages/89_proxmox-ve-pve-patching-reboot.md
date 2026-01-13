---

## Purpose

This page documents the **patching and reboot workflow for Proxmox VE (PVE)**.

PVE is treated as **hypervisor infrastructure**, not a guest system.
Patching is performed **directly on the node**, with explicit safeguards to protect storage, cluster stability, and dependent virtual machines.

Because this node hosts all VMs (including the Ansible control node), patching is **manual, deliberate, and highly visible**.

---

## Scope

### Applies to

- Proxmox VE nodes
- Systems running `pveversion`
- Hosts acting as hypervisors for all virtual infrastructure

### Explicit exclusions

This workflow does **not** apply to:

- Application VMs
- Proxmox Backup Server
- OpenMediaVault
- Bulk VM patching (`patch_all.yml`)

Those systems have dedicated workflows.

---

## Inventory Context

The Proxmox VE node is a first-class inventory host and belongs to the `infrastructure` group.

PVE is **not patched via delegation** and does **not require a `vmid`**.

---

## Entry Playbook

```text
playbooks/pve_patch.yml
```

---

## Playbook Definition

```yaml
- name: Patch and reboot Proxmox VE node
  hosts: pve
  gather_facts: yes
  become: yes

  roles:
    - role: pve_patch
      pve_patch_dry_run: false
      pve_patch_do_reboot: true
      pve_patch_enable_notify: true
```

Behavior is controlled via role variables, which may be overridden at playbook runtime.

---

## Role Structure

```text
roles/pve_patch/
├── defaults/
│   └── main.yml
└── tasks/
    ├── main.yml
    ├── precheck.yml
    ├── patch.yml
    ├── notify.yml
    └── reboot.yml
```

---

## Execution Flow (Authoritative)

### 1. Parameter Visibility

The role begins by printing key runtime parameters:

- `pve_patch_dry_run`
- `pve_patch_do_reboot`
- `pve_patch_enable_notify`

This provides immediate confirmation of **effective behavior** before any changes occur.

---

### 2. Pre-Flight Checks (`precheck.yml`)

The following safety checks are enforced:

- Proxmox tooling availability (`pveversion`)
- Current Proxmox version is displayed
- Dry-run warning is emitted if enabled
- **ZFS pool health** is checked via:
  ```bash
  zpool status -x
  ```
- Root filesystem usage is inspected
- Execution aborts if root usage exceeds ~90%

These checks prevent patching under unsafe storage conditions.

---

### 3. Patch Preparation & Upgrade (`patch.yml`)

Patch logic includes:

- Apt cache update (skipped in dry-run mode)
- Display of upgradable packages
- Full `dist-upgrade` with:
  - Autoremove
  - Autoclean
- Detection of reboot requirement via:
  ```text
  /var/run/reboot-required
  ```

Reboot requirement is **recorded for visibility**, not used as a gating condition.

---

### 4. Optional Notification (`notify.yml`)

If `pve_patch_enable_notify` is enabled:

- A detailed HTML summary is constructed
- Content includes:
  - Proxmox version
  - ZFS pool status
  - Disk usage
  - Upgradable packages
  - Reboot-required flag
- Notification is sent using the shared `notify_email` role

Notification occurs **before reboot**, while the node is still reachable.

---

### 5. Optional Reboot (`reboot.yml`)

If `pve_patch_do_reboot` is enabled:

- The node announces the impending reboot
- Reboot is executed directly on the host:
  ```bash
  /sbin/reboot
  ```
- The command is run asynchronously (`async: 1`, `poll: 0`)
- Errors are ignored to allow SSH to drop cleanly
- A short pause is introduced before exit

Key characteristics:

- Reboot is **host-initiated**, not delegated
- Reboot is **unconditional** when enabled
- The Ansible control node VM will temporarily go offline

If `pve_patch_dry_run` is enabled, reboot is skipped and logged.

---

## Pre-Flight Validation (Manual)

Before execution:

```bash
zpool status -x
df -h /
pveversion
```

Operational checks:

- No active maintenance on dependent VMs
- Ansible control node state understood
- Console access available via iDRAC / local UI

---

## Execution Commands

Normal execution:

```bash
ansible-playbook playbooks/pve_patch.yml
```

Preview execution:

```bash
ansible-playbook playbooks/pve_patch.yml --list-tasks
```

Dry run (no changes, no reboot):

```bash
ansible-playbook playbooks/pve_patch.yml -e pve_patch_dry_run=true
```

---

## Post-Patch Verification

After reboot:

```bash
pveversion
uptime -p
zpool status -x
```

Confirm:

- Proxmox services are responsive
- ZFS pools are healthy
- Virtual machines are starting normally

The Ansible control node will reconnect once its VM is back online.

---

## Failure Modes & Recovery

| Scenario | Recovery |
|--------|----------|
| Node fails to reboot | Access via console / iDRAC |
| ZFS issues detected | Abort patching, remediate storage |
| Control node unavailable | Wait for VM startup, verify network |
| SSH session drops | Expected during async reboot |

Because this is the hypervisor, recovery is **always manual**.

---

## Safety Guarantees

This workflow guarantees:

- PVE is patched in isolation
- Storage health is validated before changes
- Reboot behavior is explicit and controlled
- Notification occurs before loss of connectivity

---

## Explicit Non-Goals

This workflow does **not**:

- Patch guest VMs
- Coordinate cluster-wide rolling reboots
- Manage HA state
- Restart individual services selectively

Those concerns are handled separately.

---

## Summary

Proxmox VE patching is treated as **critical hypervisor maintenance**.

The workflow emphasizes visibility, storage safety, and deliberate reboot control.
If the node cannot safely meet these conditions, patching is deferred.

---
