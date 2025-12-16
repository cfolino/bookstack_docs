---

## Purpose

This page documents the **patching and reboot workflow for Proxmox Backup Server (PBS)**.

PBS is treated as **stateful backup infrastructure**, not a general Linux VM.
Its patching workflow prioritizes **data safety**, **task awareness**, and **controlled reboot behavior**.

Unlike bulk VM patching, PBS maintenance includes **strict pre-flight guards** and **post-reboot service validation**.

---

## Scope

### Applies to

- Proxmox Backup Server hosts
- Systems running `proxmox-backup-manager`
- Backup infrastructure providing datastore and snapshot services

### Explicit exclusions

This workflow does **not** apply to:

- Proxmox VE hosts (`pve`)
- OpenMediaVault (`omv`)
- Application VMs
- The Ansible control node

PBS is patched **independently** and never as part of `patch_all.yml`.

---

## Inventory Context

PBS is a first-class inventory host and belongs to the `infrastructure` group.

Each PBS host must define the following in host vars:

```text
inventory/host_vars/pbs.yml
```

Required variables:

```yaml
vmid: <pbs_vm_id>
```

Failure to define `vmid` will halt execution.

---

## Entry Playbook

```text
playbooks/patch_pbs.yml
```

---

## Playbook Definition

```yaml
- name: Patch Proxmox Backup Server
  hosts: pbs
  gather_facts: yes
  become: yes

  roles:
    - pbs_patch
```

PBS patching is encapsulated in a **single dedicated role**.

---

## Role Structure

```text
roles/pbs_patch/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── precheck.yml
│   ├── apt_guard.yml
│   ├── disable_man_db.yml
│   ├── patch.yml
│   ├── reboot.yml
│   └── healthcheck.yml
└── templates/
```

---

## Execution Flow (Authoritative)

### 1. Pre-Flight Safety Checks (`precheck.yml`)

Before any patching occurs, the following conditions are enforced:

- `proxmox-backup-manager` binary must exist
- OS, kernel, and host identity are logged
- **Root filesystem free space** is checked
  - Minimum required: `pbs_min_root_free_gb` (default: 3 GiB)
  - Skipped automatically in `--check` mode
- **Active PBS tasks** are queried via:
  ```bash
  proxmox-backup-manager task list
  ```
  - If tasks are running, patching **aborts**
- All checks are **hard fails** unless explicitly skipped

PBS is never patched while backup jobs are active.

---

### 2. APT & dpkg Guardrails (`apt_guard.yml`)

To prevent package manager deadlocks:

- Active `apt` / `dpkg` processes are detected
- Execution waits until locks clear
- dpkg consistency is verified
- Lock files are inspected for visibility

This ensures package operations start from a clean state.

---

### 3. Disable `man-db` Triggers (`disable_man_db.yml`)

`man-db` triggers are temporarily diverted to prevent long-running post-install hangs.

This is a **PBS-specific safeguard** based on observed behavior during upgrades.

---

### 4. OS Patching (`patch.yml`)

Patch application includes:

- Optional `apt-get --fix-broken install`
- Apt cache update
- Full `dist-upgrade` with:
  - 15-minute timeout
  - Forced config defaults (`--force-confdef`)
  - Retention of existing configs (`--force-confold`)
- Summary of package changes

Reboot-required detection is performed by checking:

```text
/var/run/reboot-required
```

Note:
- Reboot detection is recorded
- Reboot execution is handled separately

---

### 5. Hypervisor-Controlled Reboot (`reboot.yml`)

PBS is rebooted **from Proxmox**, not from the guest OS.

```yaml
delegate_to: pve
command: "qm reboot {{ vmid }}"
```

Behavior:

- Reboot is **unconditional**
- Executed on the Proxmox host
- SSH connectivity is monitored:
  - Wait for shutdown (SSH stops responding)
  - Wait for startup (SSH returns)
- No guest-initiated reboot logic is used

Proxmox availability is a **hard dependency**.

---

### 6. Post-Patch Health Verification (`healthcheck.yml`)

After reboot, the following validations are performed:

- Required PBS services are started and enabled:
  - `proxmox-backup.service`
  - `proxmox-backup-proxy.service`
- Service states are logged
- PBS package versions are retrieved
- Recent PBS task history is queried
- A structured `pbs_health_summary` fact is built

This confirms PBS is operational before declaring success.

---

### 7. Notification

A detailed patching summary is sent using the shared `notify_email` role.

The notification includes:

- Kernel version
- Free disk space
- Reboot requirement state
- Service status results
- Recent PBS task activity

---

## Pre-Flight Validation (Manual)

Before execution:

```bash
df -h /
proxmox-backup-manager task list
```

Confirm:
- Sufficient free space
- No active backup tasks
- Proxmox host reachable

---

## Execution Commands

Normal execution:

```bash
ansible-playbook playbooks/patch_pbs.yml
```

Preview execution:

```bash
ansible-playbook playbooks/patch_pbs.yml --list-tasks
```

Dry run:

```bash
ansible-playbook playbooks/patch_pbs.yml --check
```

---

## Post-Patch Verification

Recommended manual checks:

```bash
uptime -p
systemctl status proxmox-backup.service
proxmox-backup-manager versions
```

---

## Failure Modes & Recovery

| Scenario | Recovery |
|-------|----------|
| Active PBS tasks detected | Wait for completion, re-run playbook |
| Insufficient disk space | Free space, re-run playbook |
| Reboot fails | Access via Proxmox console |
| Services not running | Restart services, review logs |

Because PBS is stateful, recovery is **manual and deliberate**.

---

## Safety Guarantees

This workflow guarantees:

- PBS is never patched during active backups
- Disk capacity is validated before upgrades
- Reboot is hypervisor-controlled and observable
- Services are verified post-reboot
- Patch state is reported clearly

---

## Explicit Non-Goals

This workflow does **not**:

- Patch other systems
- Pause or reschedule backup jobs
- Perform data integrity checks
- Modify datastore configuration

Those concerns are handled externally.

---

## Summary

PBS patching is treated as **critical backup infrastructure maintenance**.

The workflow enforces strict safety gates, uses hypervisor-level reboot control, and verifies service health before completion.
If PBS cannot safely meet these conditions, patching is deferred.

---
