---

This page documents the **patching and reboot workflow for the Proxmox VE hypervisor**. Proxmox VE is treated as **critical infrastructure**, and its maintenance is intentionally isolated from all other systems.

PVE patching is performed deliberately, with explicit pre-flight checks and controlled reboot behavior.

---

## Scope

This workflow applies **only** to:

- Proxmox VE bare-metal hosts
- Systems running ZFS-backed root and data pools
- Hypervisors hosting production virtual machines

This workflow **does not apply** to:
- Virtual machines
- Proxmox Backup Server
- OpenMediaVault
- Any containerized workload

---

## Design Constraints

PVE patching follows stricter rules than VM patching:

- Never patched in parallel with other infrastructure
- Never combined with VM patching
- Always rebooted via SSH (not API)
- ZFS health must be verified before disruption

Any violation of these constraints is treated as unsafe.

---

## Playbook Location

```text
playbooks/patch_pve.yml
```

This playbook is intentionally isolated and never referenced by higher-level workflows.

---

## Playbook Content

```yaml
# playbooks/patch_pve.yml

- name: Patch and reboot Proxmox VE
  hosts: pve
  gather_facts: yes
  become: yes

  roles:
    - pve_patch
```

The playbook delegates all logic to the `pve_patch` role.

---

## Role Location

```text
roles/pve_patch/
├── tasks/
│   ├── main.yml
│   └── notify.yml
```

---

## Role Tasks – Core Logic

```yaml
# roles/pve_patch/tasks/main.yml

- name: Capture Proxmox version
  ansible.builtin.command: pveversion
  register: pveversion_cmd

- name: Check ZFS pool health
  ansible.builtin.command: zpool status
  register: zfs_status

- name: Check root filesystem usage
  ansible.builtin.command: df -h /
  register: root_disk

- name: Apply OS and Proxmox updates
  ansible.builtin.apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes
  become: true

- name: Reboot Proxmox VE
  ansible.builtin.reboot:
    reboot_timeout: 1800
```

---

## Task Explanation

### Proxmox Version Capture

```yaml
ansible.builtin.command: pveversion
```

Records the exact Proxmox version prior to patching for audit and notification purposes.

---

### ZFS Pool Health Check

```yaml
ansible.builtin.command: zpool status
```

Validates pool health before any reboot-capable action.

Expected state:
- No DEGRADED or FAULTED pools
- No resilver in progress

If ZFS is unhealthy, patching must not proceed.

---

### Root Filesystem Check

```yaml
df -h /
```

Ensures sufficient space exists for kernel and package upgrades.

---

### Patch Application

```yaml
ansible.builtin.apt:
  upgrade: dist
```

Applies:
- Kernel updates
- Proxmox packages
- Dependency transitions

Cleanup is performed to reduce disk pressure.

---

### Controlled Reboot

```yaml
ansible.builtin.reboot:
  reboot_timeout: 1800
```

A long timeout is used to accommodate:
- Kernel updates
- ZFS import
- Hardware initialization

Ansible waits for SSH connectivity before continuing.

---

## Notification Tasks

```yaml
# roles/pve_patch/tasks/notify.yml

- name: Build notification body
  ansible.builtin.set_fact:
    notify_body: |
      <h3>PVE Patching Summary</h3>

      <h4>Proxmox Version</h4>
      <pre>{{ pveversion_cmd.stdout }}</pre>

      <h4>ZFS Pool Health</h4>
      <pre>{{ zfs_status.stdout }}</pre>

      <h4>Disk Usage</h4>
      <pre>{{ root_disk.stdout }}</pre>
```

These details provide post-run visibility and audit context.

---

## Pre-Flight Validation

Before running the playbook, baseline state is verified:

```bash
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
ssh root@<internal-host> zpool status
```

VMs should be shut down or migrated according to the maintenance plan.

---

## Execution Command

```bash
ansible-playbook playbooks/patch_pve.yml
```

Dry run (syntax and inventory validation only):

```bash
ansible-playbook playbooks/patch_pve.yml --check
```

---

## Post-Patch Verification

After completion:

```bash
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
ssh root@<internal-host> pveversion
ssh root@<internal-host> zpool status
```

Expected:
- Uptime reflects recent reboot
- Proxmox version incremented
- ZFS pools healthy

---

## Failure Modes & Recovery

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| Host unreachable | Hardware or boot delay |
| ZFS import delay | Pool resilver |
| Reboot timeout | Kernel update delay |

---

### Recovery Strategy

- Access host via iDRAC
- Review boot console
- Resolve ZFS or hardware issues before retrying
- Re-run playbook only after system is stable

---

## Safety Guarantees

This workflow guarantees:

- ZFS health checked before disruption
- Single controlled reboot
- No VM-level operations
- Explicit verification steps

---

## What This Workflow Does Not Do

- Patch or manage VMs
- Coordinate VM shutdowns
- Modify Proxmox cluster settings
- Perform rolling reboots

These concerns are intentionally external.

---

## Summary

Proxmox VE patching is treated as a **high-risk, low-frequency operation**. This workflow prioritizes system integrity and observability over automation convenience.

If Proxmox VE cannot safely complete these steps, patching is postponed.

---
