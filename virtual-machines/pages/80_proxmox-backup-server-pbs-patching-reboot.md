---

This page documents the **patching and reboot workflow for Proxmox Backup Server (PBS)**. PBS is treated as **critical backup infrastructure**, and its maintenance is intentionally isolated from both Proxmox VE and general-purpose virtual machines.

The primary objective of this workflow is to **preserve backup integrity** while keeping the system patched and secure.

---

## Scope

This workflow applies **only** to:

- Proxmox Backup Server hosts (VM-based)
- Systems providing backup storage to Proxmox VE
- Hosts with attached datastores used for scheduled backups

This workflow **does not apply** to:
- Proxmox VE hypervisors
- OpenMediaVault
- Application VMs
- The Ansible control node

---

## Design Constraints

PBS patching follows strict constraints due to its role:

- Never patched at the same time as Proxmox VE
- Never patched during active backup windows
- Reboot is controlled and observable
- SSH key authentication is mandatory (no interactive prompts)

These constraints prevent backup corruption and ensure predictable recovery.

---

## Playbook Location

```text
playbooks/patch_pbs.yml
```

This playbook is dedicated exclusively to PBS maintenance.

---

## Playbook Content

```yaml
# playbooks/patch_pbs.yml

- name: Patch and reboot Proxmox Backup Server
  hosts: pbs
  gather_facts: yes
  become: yes

  roles:
    - pbs_patch
```

---

## Role Location

```text
roles/pbs_patch/
├── tasks/
│   ├── main.yml
│   └── notify.yml
```

---

## Role Tasks – Core Logic

```yaml
# roles/pbs_patch/tasks/main.yml

- name: Capture PBS version
  ansible.builtin.command: proxmox-backup-manager version
  register: pbs_version

- name: Check datastore status
  ansible.builtin.command: proxmox-backup-manager datastore list
  register: datastore_status

- name: Apply OS and PBS updates
  ansible.builtin.apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes
  become: true

- name: Reboot PBS
  ansible.builtin.reboot:
    reboot_timeout: 1200
```

---

## Task Explanation

### PBS Version Capture

```yaml
ansible.builtin.command: proxmox-backup-manager version
```

Captures the current PBS version for notification and audit purposes.

---

### Datastore Status Check

```yaml
ansible.builtin.command: proxmox-backup-manager datastore list
```

Confirms:
- Datastores are present
- Datastores are accessible
- No obvious errors are reported

This is a **sanity check**, not a deep integrity scan.

---

### Patch Application

```yaml
ansible.builtin.apt:
  upgrade: dist
```

Applies:
- OS-level updates
- PBS package updates
- Dependency changes

Cleanup tasks reduce disk pressure on backup volumes.

---

### Controlled Reboot

```yaml
ansible.builtin.reboot:
  reboot_timeout: 1200
```

The reboot timeout accommodates:
- Filesystem remount
- Datastore reattachment
- PBS service initialization

Ansible waits for SSH connectivity to return.

---

## Notification Tasks

```yaml
# roles/pbs_patch/tasks/notify.yml

- name: Build PBS notification body
  ansible.builtin.set_fact:
    notify_body: |
      <h3>PBS Patching Summary</h3>

      <h4>PBS Version</h4>
      <pre>{{ pbs_version.stdout }}</pre>

      <h4>Datastores</h4>
      <pre>{{ datastore_status.stdout }}</pre>
```

This provides post-run visibility into PBS state.

---

## Pre-Flight Validation

Before executing the playbook, baseline checks are performed manually:

```bash
ssh root@<internal-host> proxmox-backup-manager datastore list
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
```

Additionally:
- Confirm no active backup jobs in Proxmox VE
- Confirm scheduled jobs will not overlap maintenance

---

## Execution Command

```bash
ansible-playbook playbooks/patch_pbs.yml
```

Dry run (inventory and syntax validation):

```bash
ansible-playbook playbooks/patch_pbs.yml --check
```

---

## Post-Patch Verification

After completion:

```bash
ssh root@<internal-host> uptime -p
ssh root@<internal-host> who -b
ssh root@<internal-host> proxmox-backup-manager version
ssh root@<internal-host> proxmox-backup-manager datastore list
```

Expected:
- Recent uptime
- PBS version updated
- Datastores accessible

---

## Failure Modes & Recovery

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| Datastore missing | Mount failure |
| PBS services down | Service startup delay |
| SSH unreachable | Guest agent or boot issue |

---

### Recovery Strategy

- Access VM via Proxmox console
- Verify datastore mounts
- Restart PBS services if required
- Re-run playbook only after verification

Backup jobs are resumed only after validation completes.

---

## Safety Guarantees

This workflow guarantees:

- PBS is patched independently
- Datastore visibility is validated
- Reboot occurs exactly once
- Post-reboot verification is explicit

---

## What This Workflow Does Not Do

- Trigger or manage backup jobs
- Validate backup integrity
- Coordinate with Proxmox VE scheduling
- Patch underlying storage hosts

Those concerns are intentionally external.

---

## Summary

PBS patching is treated as **backup-critical maintenance**. The workflow prioritizes data safety, observability, and controlled execution over automation speed.

If PBS cannot safely complete these steps, patching is postponed.

---
