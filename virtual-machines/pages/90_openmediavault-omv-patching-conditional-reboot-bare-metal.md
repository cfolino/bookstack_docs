## Purpose

This page documents the **patching and reboot workflow for OpenMediaVault (OMV)**.

OMV is treated as **stateful storage infrastructure**. The patching workflow is designed to be safe, observable, and minimally disruptive, with:

- Clear pre-patch baseline recording
- Optional read-only Btrfs health visibility
- Conditional reboot logic (only when required)
- Graceful shutdown of Samba services prior to reboot
- Post-reboot connectivity and sanity checks
- Email reporting via shared notification infrastructure

Unlike VM workflows that are restarted from Proxmox, OMV is patched and rebooted as a **bare-metal host**, using Ansible’s built-in `reboot` module.

---

## Scope

### Applies to

- OpenMediaVault bare-metal host(s) targeted by inventory group `omv`
- Systems patched via APT using `dist-upgrade`

### Explicit exclusions

This workflow does **not** apply to:

- Proxmox VE hosts (`pve`)
- Proxmox Backup Server (`pbs`)
- Bulk Ubuntu VM orchestration (`patch_all.yml`)
- Ansible control node patching (`patch_ansible_node.yml`)

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
      ansible.builtin.set_fact:
        omv_email_body: |
          <h2>OMV Patch &amp; Reboot Report</h2>
          <pre>{{ omv_patch_summary }}</pre>
          <h3>Btrfs Status</h3>
          <pre>{{ omv_btrfs_summary }}</pre>

    - name: Send OMV patch notification email
      ansible.builtin.include_role:
        name: notify_email
      vars:
        notify_subject: "OMV Patching Completed ({{ inventory_hostname }})"
        notify_body: "{{ omv_email_body }}"
```

The email report is assembled from facts accumulated during the role execution.

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

- Gathering system facts (`setup`)
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
  - `omv_btrfs_summary` is set to “No Btrfs mounts detected on OMV.”
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
- Full OS upgrade via `dist-upgrade`
  - `autoremove` and `autoclean` enabled
- Detects reboot-required marker:
  ```text
  /var/run/reboot-required
  ```

Updates `omv_patch_summary` with:

- Whether apt reported changes (`omv_apt_upgrade.changed`)
- Whether reboot-required file exists

---

### 4. Conditional Reboot &amp; Service Handling (`reboot.yml`)

A reboot decision is made:

```yaml
omv_should_reboot: >
  reboot-required file exists OR apt upgrade changed
```

This means a reboot occurs when:

- `/var/run/reboot-required` exists, **or**
- The package upgrade actually changed the system

#### If reboot is NOT required

- A message is logged:
  - “No reboot required for OMV.”
- No reboot-related tasks execute.

#### If reboot IS required

The following actions occur:

1. Collect service facts
2. Build a list of Samba services present:
   - Checks for `samba` and `smbd` in `ansible_facts.services`
3. Gracefully stop Samba services (if present)
4. Append Samba stop actions into `omv_patch_summary`
5. Run a pre-reboot ping check to the configured ping host
6. Reboot OMV using Ansible’s reboot module (bare metal):
   ```yaml
   ansible.builtin.reboot:
     reboot_timeout: "{{ omv_reboot_timeout }}"
     test_command: "whoami"
   ```
7. Wait for SSH to return (delegated to `localhost`)
8. Run post-reboot ping check
9. Capture post-reboot uptime
10. Append reboot details and post-reboot uptime into `omv_patch_summary`

Important characteristics:

- Reboot is **guest / bare-metal initiated**, not Proxmox-controlled
- Samba is stopped **only when rebooting**, and only if services exist
- Connectivity is validated after reboot before declaring success

---

## Defaults &amp; Tunables

From `roles/omv_patch/defaults/main.yml`:

```yaml
omv_reboot_timeout: 900
omv_ping_host: "{{ ansible_host }}"
omv_ping_count: 3
omv_ping_interval: 5
omv_btrfs_checks_enabled: true
```

Notes:

- No `inventory/host_vars/omv.yml` currently exists
- Behavior is driven by role defaults and inventory host definition
- Ping host defaults to `ansible_host`

---

## Pre-Flight Validation (Manual)

Before execution, confirm:

```bash
df -h /
uptime -p
systemctl status smbd --no-pager || true
systemctl status samba --no-pager || true
```

If OMV provides active file services, prefer running patching during a maintenance window.

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

## Failure Modes &amp; Recovery

| Scenario | Likely Cause | Recovery |
|--------|-------------|----------|
| Reboot hangs | Hardware or service shutdown delay | Access via console / iDRAC, review logs |
| SSH does not return | Network or interface issues | Console access, verify IP and services |
| Samba does not return | Service stopped pre-reboot | Start services manually, verify config |
| Btrfs stats fail | Tool unavailable or mount inaccessible | Informational only; review manually |

---

## Safety Guarantees

This workflow guarantees:

- Pre- and post-patch observability
- Reboots only when required
- Graceful Samba shutdown before reboot (if present)
- Post-reboot connectivity verification
- Structured email reporting for traceability

---

## Explicit Non-Goals

This workflow does **not**:

- Explicitly restart Samba services post-reboot
- Force reboots when no changes occurred
- Perform filesystem scrubs or repairs
- Patch unrelated infrastructure systems

---

## Summary

OMV patching is treated as **stateful infrastructure maintenance**.

The workflow records baseline state, applies upgrades, performs optional read-only Btrfs visibility checks, and reboots only when required — with service-aware handling and post-reboot validation.

---
