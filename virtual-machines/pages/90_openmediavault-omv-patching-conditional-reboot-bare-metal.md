---

## Purpose

This page documents the **patching and reboot workflow for OpenMediaVault (OMV)**.

OMV is treated as **stateful storage infrastructure**. The patching workflow is designed to be safe and observable, with:

- Clear pre-patch baseline recording
- Optional read-only Btrfs health visibility
- Conditional reboot logic (only when required)
- Graceful shutdown of Samba services before reboot
- Post-reboot connectivity and sanity checks
- Email reporting via shared notification infrastructure

Unlike VM workflows that restart from Proxmox, OMV is patched and rebooted as a **bare-metal host**, using Ansible’s reboot module.

---

## Scope

### Applies to

- OMV bare-metal host(s) targeted by inventory group `omv`
- Systems patched via APT (`dist-upgrade`)

### Explicit exclusions

This workflow does **not** apply to:

- Proxmox VE hosts (`pve`)
- Proxmox Backup Server (`pbs`)
- Bulk Ubuntu VM orchestration (`patch_all.yml`)
- Ansible control node (`patch_ansible_node.yml`)

---

## Entry Playbook

```text
playbooks/patch_omv.yml
```

---

## Playbook Definition (Authoritative)

```yaml
- name: Patch and reboot OMV bare-metal
  hosts: omv
  gather_facts: yes
  become: yes

  roles:
    - omv_patch

  post_tasks:

    - name: Build OMV patch email body
      set_fact:
        omv_email_body: |
          <h2>OMV Patch & Reboot Report</h2>
          <pre>{{ omv_patch_summary }}</pre>
          <h3>Btrfs Status</h3>
          <pre>{{ omv_btrfs_summary }}</pre>

    - name: Send OMV patch notification email
      include_role:
        name: notify_email
      vars:
        notify_subject: "OMV Patching Completed ({{ inventory_hostname }})"
        notify_body: "{{ omv_email_body }}"
```

The email summary is built from facts accumulated during the role run.

---

## Role Structure

```text
roles/omv_patch/
├── defaults/
│   └── main.yml
└── tasks/
    ├── main.yml
    ├── prechecks.yml
    ├── btrfs.yml
    ├── patch.yml
    └── reboot.yml
```

---

## Execution Flow (Authoritative)

### 1. Prechecks (`prechecks.yml`)

The role begins by:

- Gathering facts (`setup`)
- Capturing pre-patch baseline information:
  - OS (from `/etc/os-release`)
  - Uptime (`uptime -p`)

A running summary fact is initialized:

- `omv_patch_summary`

This becomes the basis of the final email report.

---

### 2. Optional Btrfs Health Checks (`btrfs.yml`)

Btrfs checks run only when enabled:

```yaml
when: omv_btrfs_checks_enabled | bool
```

Behavior:

- Detects Btrfs mounts from `ansible_facts.mounts`
- If no Btrfs mounts exist:
  - `omv_btrfs_summary` is set to “No Btrfs mounts detected…”
- If Btrfs mounts exist:
  - Runs read-only command per mount:
    ```bash
    btrfs device stats <mount>
    ```
  - Builds a formatted summary (`omv_btrfs_summary`)

Important notes:

- These checks are **read-only**
- Failures are tolerated (`failed_when: false`) to avoid blocking patching

---

### 3. OS Patching (`patch.yml`)

Patch phase performs:

- Apt cache update (cached for 1 hour)
- Full OS upgrade via `dist-upgrade`:
  - autoremove + autoclean enabled
- Detects reboot-required marker:
  ```text
  /var/run/reboot-required
  ```

Updates `omv_patch_summary` with:

- Whether apt reported changes (`omv_apt_upgrade.changed`)
- Whether reboot-required file exists

---

### 4. Conditional Reboot + Service Handling (`reboot.yml`)

A reboot decision is made:

```yaml
omv_should_reboot: >
  reboot-required file exists OR apt upgrade changed
```

This means reboot occurs when:
- `/var/run/reboot-required` exists, **or**
- package upgrade actually changed the system

#### If reboot is NOT required
- A message is logged:
  - “No reboot required for OMV.”
- No further reboot tasks run.

#### If reboot IS required
The following actions occur:

1) Collect service facts
2) Build list of Samba services present:
   - checks for `samba` and `smbd` keys in `ansible_facts.services`
3) Gracefully stop Samba services (if present)
4) Append Samba stop actions into `omv_patch_summary`
5) Run a pre-reboot ping check to the configured ping host
6) Reboot OMV using Ansible’s reboot module (bare metal):
   ```yaml
   ansible.builtin.reboot:
     reboot_timeout: "{{ omv_reboot_timeout }}"
     test_command: "whoami"
   ```
7) Wait for SSH to return (delegated to localhost)
8) Post-reboot ping check
9) Capture post-reboot uptime
10) Append reboot details and post-uptime into `omv_patch_summary`

Important characteristics:

- Reboot is **guest/bare-metal initiated**, not Proxmox-controlled
- Samba is stopped **only when rebooting**, and only if services exist
- Connectivity is validated after reboot before declaring success

---

## Defaults & Tunables

From `roles/omv_patch/defaults/main.yml`:

```yaml
omv_reboot_timeout: 900
omv_ping_host: "{{ ansible_host }}"
omv_ping_count: 3
omv_ping_interval: 5
omv_btrfs_checks_enabled: true
```

Notes:

- No `inventory/host_vars/omv.yml` exists currently; behavior is driven by role defaults and inventory host definition.
- Ping host defaults to `ansible_host`.

---

## Pre-Flight Validation (Manual)

Before execution, confirm:

```bash
df -h /
uptime -p
systemctl status smbd --no-pager || true
systemctl status samba --no-pager || true
```

If OMV provides active file services, prefer running patching in a maintenance window.

---

## Execution Commands

Normal execution:

```bash
ansible-playbook playbooks/patch_omv.yml
```

Preview execution:

```bash
ansible-playbook playbooks/patch_omv.yml --list-tasks
```

Dry run:

```bash
ansible-playbook playbooks/patch_omv.yml --check
```

---

## Post-Patch Verification

After completion (and reboot if performed):

```bash
uptime -p
who -b
systemctl status smbd --no-pager || true
systemctl status samba --no-pager || true
```

If Btrfs is in use:

```bash
btrfs device stats /
```

(or run against the relevant mountpoints)

---

## Failure Modes & Recovery

| Scenario | Likely Cause | Recovery |
|---|---|---|
| Reboot hangs | Hardware / service shutdown delay | Access via console/iDRAC, review logs |
| SSH does not return | Network / interface issues | Console access, verify IP + services |
| Samba does not return | Service stop succeeded but not restarted automatically | Start services manually, verify config |
| Btrfs stats fail | Tool unavailable or mount not accessible | Treat as informational; review manually |

---

## Safety Guarantees

This workflow guarantees:

- Patching is performed with pre/post observability
- Reboot occurs only when required
- Samba is stopped gracefully prior to reboot (if present)
- Post-reboot connectivity is verified
- A structured email report is generated for traceability

---

## Explicit Non-Goals

This workflow does **not**:

- Restart Samba services after reboot explicitly (system boot handles service startup)
- Force reboots if no changes occurred
- Perform deep filesystem scrubs or repairs
- Patch other infrastructure systems

---

## Summary

OMV patching is treated as **stateful infrastructure maintenance**.

The workflow records baseline state, applies upgrades, performs optional read-only Btrfs visibility checks, and only reboots when required — with service-aware handling and post-reboot validation.

---
