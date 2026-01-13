## Purpose

This page documents the **patching and reboot workflow for the Ansible control node**.

The Ansible control node is treated as **automation control-plane infrastructure**, not a general-purpose system. Because it orchestrates patching, reboots, backups, and configuration changes across the environment, it is patched **last**, **in isolation**, and with **maximum observability**.

Loss or instability of this node would interrupt all automation, so this workflow prioritizes **state capture, deterministic execution, repository consistency, and enforced post-reboot validation**.

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

    @infrastructure:
      ├── pve
      ├── pbs
      ├── ansible
      └── omv

Isolation is achieved through **explicit playbook targeting**, not inventory separation.

---

## Design Constraints

This workflow follows strict rules:

- Never executed concurrently with other patching jobs
- Always patched last
- Executed via automation (Semaphore or direct Ansible invocation)
- GitHub is the authoritative source of automation code
- Reboot authority belongs to the hypervisor
- Control-plane health is enforced after reboot

Automation infrastructure is required to be **more stable than the systems it manages**.

---

## Entry Playbook

    playbooks/patch_ansible_node.yml

---

## Playbook Definition

    - name: Patch and reboot Ansible control node
      hosts: ansible
      gather_facts: yes
      become: yes

      vars:
        ansible_node_git_pull: true

      roles:
        - ansible_node_patch

Notes:

- A **dedicated role** is used
- No shared patching roles are included
- The `ansible_node_git_pull` variable explicitly controls repository synchronization

---

## Role Structure

    roles/ansible_node_patch/
    └── tasks/
        └── main.yml

This role fully encapsulates all logic for control-plane patching, backup, shutdown, notification, reboot, repository synchronization, and post-reboot validation.

---

## Execution Flow (Authoritative)

### 1. Baseline System State Capture

The following information is collected **automatically before any changes**:

- OS details
- Kernel version (before)
- Uptime (before)
- Root filesystem usage (before)
- Semaphore service status (before)
- Local Git repository status of `/home/ansible/ansible`

These tasks are **read-only** and always executed.

---

### 2. Repository Synchronization (Conditional)

If `ansible_node_git_pull` is set to `true`:

- The local automation repository at `/home/ansible/ansible` is updated from the configured remote Git source
- The control node aligns itself with the authoritative codebase used by Semaphore
- Repository updates are explicit and intentional

If the variable is set to `false`, the working tree is left untouched.

No repository updates occur implicitly.

---

### 3. Package Preparation

- Apt cache is updated (with cache validity enforcement)
- List of upgradable packages is captured prior to upgrade

---

### 4. Control-Plane Backup

A local backup archive is created containing:

- SSH keys
- Semaphore systemd unit files
- Semaphore override files
- Local configuration JSON

The archive is stored under:

    /home/ansible/backups/

Permissions are corrected to restrict access to the `ansible` user.

This backup is performed **every run**.

---

### 5. OS Upgrade

- Full distribution upgrade is applied
- Autoremove and autoclean are executed
- List of upgradable packages is captured again after upgrade

---

### 6. State Comparison

After upgrades:

- Kernel version is captured again
- Kernel change is detected and recorded
- Root filesystem usage is captured again

Kernel change is **observed**, but does **not** gate reboot behavior.

---

### 7. Automation Shutdown

Semaphore is stopped **explicitly** prior to reboot.

Service status is captured after shutdown to preserve observability.

Automation execution is intentionally halted before system restart.

---

### 8. Notification

A detailed HTML patching summary email is constructed and sent via the shared notification role.

The notification includes:

- OS details
- Kernel before and after
- Disk usage before and after
- Git repository status
- Semaphore service status
- Apt upgrade results
- Backup archive path
- Reboot note

Notification delivery occurs **before reboot** to avoid post-reboot loss.

---

### 9. Reboot (Critical)

The control node is rebooted **from the hypervisor**, not from the guest OS.

Key characteristics:

- Reboot is unconditional
- Executed on the Proxmox host
- Requires the VMID defined in host variables
- Does not use in-guest reboot mechanisms

Proxmox availability is a hard dependency.

---

### 10. Post-Reboot Control-Plane Validation

After the VM returns from reboot:

- Ansible waits for SSH connectivity to be restored
- Semaphore service is started if not already running
- Semaphore service state is explicitly validated
- The job fails if control-plane health cannot be restored

This guarantees that the automation scheduler remains functional after maintenance.

---

## Failure Modes & Recovery

If the workflow fails after reboot:

- The VM console is accessed via Proxmox
- Filesystem and service health are verified manually
- Semaphore may be restarted manually if required

Because this system is the automation control plane, recovery is always **deliberate and manual**.

---

## Safety Guarantees

This workflow guarantees:

- Control-plane isolation
- Explicit backup before upgrade
- Repository state awareness
- Automation shutdown prior to reboot
- Hypervisor-controlled reboot
- Enforced post-reboot scheduler health

---

## Summary

The Ansible control node is patched conservatively and intentionally.

This workflow reflects the principle that **automation infrastructure must be more stable than the systems it manages**, and that both **code and execution state must be explicitly controlled** during maintenance.
