---
This page documents the **patching and reboot workflow for OpenMediaVault (OMV)**. OMV is treated as **stateful storage infrastructure**, not a generic Linux host. Its maintenance is deliberately isolated to protect data availability and integrity.

Unlike application VMs, OMV patching is performed with heightened caution and within a dedicated maintenance window.

---

## Scope

This workflow applies **only** to:

- OpenMediaVault hosts
- Systems providing NAS, CIFS/NFS, or backup storage
- Hosts using non-ZFS filesystems (ext4 in this environment)

This workflow **does not apply** to:
- Proxmox VE
- Proxmox Backup Server
- Application virtual machines
- The Ansible control node

---

## Design Constraints

OMV patching follows strict constraints:

- Never patched alongside hypervisors or VMs
- Never patched during active backup or file transfer windows
- Services must be allowed to shut down cleanly
- Reboot is explicit and observable

OMV is treated as a **single point of failure for storage**, and automation reflects that risk.

---

## Playbook Location

```text
playbooks/patch_omv.yml
```

This playbook is dedicated exclusively to OMV maintenance.

---

## Playbook Content

```yaml
# playbooks/patch_omv.yml

- name: Patch and reboot OpenMediaVault
  hosts: omv
  gather_facts: yes
  become: yes

  roles:
    - omv_patch
```

---

## Role Location

```text
roles/omv_patch/
├── tasks/
│   ├── main.yml
│   └── notify.yml
```

---

## Role Tasks – Core Logic

```yaml
# roles/omv_patch/tasks/main.yml

- name: Capture OMV version
  ansible.builtin.command: omv-version
  register: omv_version

- name: Check root filesystem usage
  ansible.builtin.command: df -h /
  register: root_disk

- name: Apply OS and OMV updates
  ansible.builtin.apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes
  become: true

- name: Reboot OMV
  ansible.builtin.reboot:
    reboot_timeout: 1200
```

---

## Task Explanation

### OMV Version Capture

```yaml
ansible.builtin.command: omv-version
```

Records the OMV version prior to patching for audit and notification purposes.

---

### Root Filesystem Check

```yaml
df -h /
```

Ensures sufficient space exists for OS and OMV package upgrades.

---

### Patch Application

```yaml
ansible.builtin.apt:
  upgrade: dist
```

Applies:
- Debian OS updates
- OpenMediaVault packages
- Dependency transitions

Cleanup tasks reduce disk usage and stale packages.

---

### Controlled Reboot

```yaml
ansible.builtin.reboot:
  reboot_timeout: 1200
```

The reboot timeout accommodates:
- Service shutdown
- Filesystem remount
- Storage stack initialization

Ansible waits for SSH connectivity before completing the run.

---

## Notification Tasks

```yaml
# roles/omv_patch/tasks/notify.yml

- name: Build OMV notification body
  ansible.builtin.set_fact:
    notify_body: |
      <h3>OMV Patching Summary</h3>

      <h4>OMV Version</h4>
      <pre>{{ omv_version.stdout }}</pre>

      <h4>Disk Usage</h4>
      <pre>{{ root_disk.stdout }}</pre>
```

These details provide post-maintenance visibility and audit context.

---

## Pre-Flight Validation

Before executing the playbook, baseline state is verified manually:

```bash
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
ssh root@<internal-host> df -h /
```

Operational checks:
- Confirm no active backups or file transfers
- Confirm dependent systems are idle or paused

---

## Execution Command

```bash
ansible-playbook playbooks/patch_omv.yml
```

Dry run (syntax and inventory validation):

```bash
ansible-playbook playbooks/patch_omv.yml --check
```

---

## Post-Patch Verification

After completion:

```bash
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
ssh root@<internal-host> omv-version
ssh root@<internal-host> systemctl status omv-engined
```

Expected:
- Recent uptime
- OMV services active
- No filesystem errors

---

## Failure Modes & Recovery

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| Services not starting | Delayed OMV engine |
| SSH unreachable | Boot delay |
| Filesystems missing | Mount or disk issue |

---

### Recovery Strategy

- Access host via console or iDRAC
- Verify disk mounts
- Restart OMV services if required
- Re-run playbook only after system is stable

---

## Safety Guarantees

This workflow guarantees:

- Storage patched independently
- Single controlled reboot
- Explicit verification steps
- No interaction with VMs or hypervisors

---

## What This Workflow Does Not Do

- Manage backup jobs
- Validate data integrity
- Modify share configuration
- Patch dependent clients

These concerns are intentionally external.

---

## Summary

OMV patching is treated as **stateful storage maintenance**, not routine OS upkeep. The workflow emphasizes caution, isolation, and explicit validation to protect stored data.

If OMV cannot safely complete these steps, patching is deferred.

---
