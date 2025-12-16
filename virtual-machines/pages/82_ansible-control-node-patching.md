---

## Purpose

This page documents the **patching and reboot workflow for the Ansible control node**.

The Ansible control node is treated as **automation control-plane infrastructure**, not a general-purpose system. Because it orchestrates patching, reboots, backups, and configuration changes across the environment, it is patched **last**, **in isolation**, and with **maximum observability**.

Loss or instability of this node would interrupt all automation, so this workflow prioritizes **safety, state capture, and controlled reboot behavior**.

---

## Scope

This workflow applies **only** to:

- The Ansible control node
- The system hosting:
  - Playbooks
  - Inventory
  - SSH keys
  - Semaphore (job orchestration)
- The VM identified as `ansible` in inventory

This workflow **does not apply** to:

- Application virtual machines
- Proxmox VE hosts
- Proxmox Backup Server
- OpenMediaVault
- Bulk or orchestrated patching workflows

---

## Inventory Context

The Ansible control node is a **first-class inventory host**, not `localhost`.

It belongs to the `infrastructure` group:

```text
@infrastructure:
  ├── pve
  ├── pbs
  ├── ansible
  └── omv
```

Isolation is achieved through **explicit playbook targeting**, not inventory separation.

---

## Design Constraints

This workflow follows strict rules:

- Never executed concurrently with other patching jobs
- Never executed while Semaphore jobs are running
- Always executed manually
- Always patched last
- Reboot must be explicit and observable

Automation infrastructure is required to be **more stable than the systems it manages**.

---

## Entry Playbook

```text
playbooks/patch_ansible_node.yml
```

---

## Playbook Definition

```yaml
- name: Patch and reboot Ansible control node
  hosts: ansible
  gather_facts: yes
  become: yes

  vars:
    ansible_node_git_pull: true

  roles:
    - ansible_node_patch
```

Notes:

- A **dedicated role** is used
- No shared patching roles are included
- The `ansible_node_git_pull` variable is currently unused and has no effect

---

## Role Structure

```text
roles/ansible_node_patch/
└── tasks/
    └── main.yml
```

This role fully encapsulates all logic for control-plane patching, backup, shutdown, notification, and reboot.

---

## Execution Flow (Authoritative)

### 1. Baseline System State Capture

The following information is collected **before any changes**:

- OS details (`lsb_release`)
- Kernel version (before)
- Uptime (before)
- Root filesystem usage (before)
- Semaphore service status (before)
- Git repository status of `/home/ansible/ansible`

These tasks are **read-only** and always executed.

---

### 2. Package Preparation

- Apt cache is updated (with `cache_valid_time`)
- List of upgradable packages is captured (before upgrade)

---

### 3. Control-Plane Backup

A local backup archive is created containing:

- `/home/ansible/.ssh`
- Semaphore systemd unit files
- Semaphore overrides
- Local configuration JSON

The archive is stored under:

```text
/home/ansible/backups/
```

Permissions are corrected to restrict access to the `ansible` user.

This backup is performed **every run**.

---

### 4. OS Upgrade

- Full `dist-upgrade` is applied
- Autoremove and autoclean are executed
- List of upgradable packages is captured again (after)

---

### 5. State Comparison

After upgrades:

- Kernel version is captured again
- Kernel change is detected and recorded
- Root filesystem usage is captured again

Important:

- Kernel change is **detected**
- Kernel change **does not gate reboot behavior**

---

### 6. Automation Shutdown

Semaphore is stopped **explicitly**:

```yaml
systemd:
  name: semaphore.service
  state: stopped
```

- Service status is captured after stop
- Semaphore is **not restarted** in this workflow
- Automation resumes only after the VM returns from reboot

---

### 7. Notification

A detailed **HTML patching summary email** is constructed and sent via the shared `notify_email` role.

The notification includes:

- OS details
- Kernel before/after
- Disk usage before/after
- Git repository status
- Semaphore service status
- Apt upgrade results
- Backup archive path
- Reboot note

The notification logic itself is centralized and not reimplemented here.

---

### 8. Reboot (Critical)

The control node is rebooted **from the hypervisor**, not from the guest OS.

```yaml
delegate_to: pve
command: "qm reboot {{ vmid }}"
```

Key characteristics:

- Reboot is **unconditional**
- Executed on the Proxmox host (`pve`)
- Requires `vmid` defined in:

```text
inventory/host_vars/ansible.yml
```

- Does **not** use:
  - `ansible.builtin.reboot`
  - Handler-based reboots
  - Shared reboot orchestration roles

Proxmox availability is a **hard dependency**.

---

## Pre-Flight Validation

Before execution, the following checks are performed manually on the control node:

```bash
uptime -p
who -b
df -h /
```

Operational confirmation:

- No Semaphore jobs running
- No active Ansible playbooks
- Proxmox host reachable

If any condition is not met, patching is deferred.

---

## Execution Command

```bash
ansible-playbook playbooks/patch_ansible_node.yml
```

Dry run (syntax and resolution only):

```bash
ansible-playbook playbooks/patch_ansible_node.yml --check
```

---

## Post-Patch Verification

After the VM returns:

```bash
uptime -p
who -b
ansible --version
```

Expected:

- Recent uptime
- Control node reachable
- Ansible runtime intact

Semaphore jobs are resumed manually after verification.

---

## Failure Modes & Recovery

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| SSH unavailable | VM still rebooting |
| Semaphore unreachable | Service not yet restarted |
| Reboot fails | Proxmox connectivity issue |

---

### Recovery Strategy

- Access VM console via Proxmox
- Verify disk and filesystem health
- Restart Semaphore manually if required
- Re-run playbook only after system stability is confirmed

Because this is the control plane, recovery is always **manual and deliberate**.

---

## Safety Guarantees

This workflow guarantees:

- Control-plane isolation
- Explicit backup before upgrade
- Automation shutdown prior to reboot
- Hypervisor-controlled reboot
- Immediate visibility through notification

---

## Explicit Non-Goals

This workflow does **not**:

- Patch other hosts
- Restart Semaphore automatically
- Perform repository updates
- Use shared patching or reboot roles

Those concerns are handled elsewhere.

---

## Summary

The Ansible control node is patched conservatively and intentionally.

This workflow reflects the principle that **automation infrastructure must be more stable than the systems it manages**, and that reboot authority belongs to the hypervisor, not the guest.

If the control node cannot safely meet these conditions, patching is postponed.

---
