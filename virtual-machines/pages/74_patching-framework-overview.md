---

This page documents the **design, intent, and operational structure** of the Ansible-based patching and reboot framework I built for the `cfolino.com` homelab. It serves as the authoritative reference for how patching is performed, why it is structured the way it is, and how new systems are safely integrated into the workflow.

The goal of this framework is not convenience — it is **predictability, safety, and auditability**.

---

## Purpose

This patching framework provides **controlled, repeatable OS maintenance** across a mixed environment consisting of:

- Ubuntu virtual machines
- Bare-metal hypervisors
- Backup infrastructure
- Stateful storage systems
- The Ansible control node itself

I intentionally avoided a single “patch everything” workflow. Instead, patching is divided into **clearly defined tiers** to prevent cascading outages and circular dependencies.

---

## High-Level Architecture

| Component | Responsibility |
|---------|----------------|
| Ansible Control Node | Executes all automation |
| Inventory | Defines scope and tiering |
| Roles | Encapsulate patch and reboot logic |
| Semaphore (optional) | Scheduling and RBAC |
| Notifications | Visibility and audit trail |

All automation is executed from the Ansible control node. Semaphore is layered on top for scheduling and access control but does not alter playbook behavior.

---

## Supported System Classes

The framework currently supports the following system classes:

| Class | Examples |
|-----|---------|
| Ubuntu VMs | Grafana, BookStack, Pi-hole |
| Hypervisor | Proxmox VE |
| Backup | Proxmox Backup Server |
| Storage | OpenMediaVault |
| Control Plane | Ansible node |

Each class is patched independently and never combined in a single run.

---

## Adding a New Host to the Framework

Before any patching occurs, a new host must be explicitly integrated and validated.

### 1. Verify SSH Access

From the Ansible control node:

```bash
ssh ansible@NEW_HOST_FQDN
```

Key-based SSH access is a hard requirement. If this fails, the host is not added.

---

### 2. Add Host to Inventory

Edit the inventory file:

```bash
nano ~/ansible/inventory/hosts.yml
```

Add the host to the appropriate group:

```yaml
ubuntu_vms:
  hosts:
    newhost:
      ansible_host: <internal-host>
```

The group selection determines which patching logic applies.

---

### 3. Validate Ansible Connectivity

```bash
ansible newhost -m ping
```

Successful output confirms inventory and SSH configuration.

---

### 4. Perform a Dry Run

Before allowing any changes:

```bash
ansible-playbook playbooks/patch_all.yml --limit newhost --check
```

This validates:
- Inventory resolution
- Role inclusion
- Syntax and variable correctness

---

## Core Design Principles

### No Blind Reboots

Reboots are never implicit or duplicated.
A system reboots **only** when the operating system explicitly requires it.

Verification before patching:

```bash
ansible newhost -a "test -f /var/run/reboot-required && echo REBOOT_REQUIRED || echo NO_REBOOT"
```

All reboot logic is centralized in a single role.

---

### Explicit Tier Separation

I enforce strict tier separation to avoid cascading failures.

Examples of valid executions:

```bash
ansible-playbook playbooks/patch_all.yml        # Ubuntu VMs
ansible-playbook playbooks/patch_pve.yml        # Proxmox VE
ansible-playbook playbooks/patch_pbs.yml        # PBS
ansible-playbook playbooks/patch_omv.yml        # OMV
ansible-playbook playbooks/patch_ansible.yml    # Control node
```

No playbook spans multiple tiers.

---

### Mandatory Pre-Flight Validation

Before patching any host, I verify baseline state:

```bash
ansible TARGET -m ping
ansible TARGET -a "df -h /"
ansible TARGET -a "uptime -p"
ansible TARGET -a "who -b"
```

For virtual machines:

```bash
ansible TARGET -a "systemctl status qemu-guest-agent"
```

If these checks fail, patching does not proceed.

---

## Executing Patching Workflows

### Ubuntu Virtual Machines

```bash
ansible-playbook playbooks/patch_all.yml
```

This workflow:
1. Ensures the QEMU Guest Agent is running
2. Applies OS updates
3. Reboots only if required
4. Performs service-specific health checks
5. Sends notifications

---

### Proxmox VE

```bash
ansible-playbook playbooks/patch_pve.yml
```

This includes:
- Proxmox version capture
- ZFS pool health validation
- Controlled reboot via SSH

---

### Proxmox Backup Server

```bash
ansible-playbook playbooks/patch_pbs.yml
```

I verify that:
- No backup jobs are running
- Datastores are healthy

before rebooting the system.

---

### OpenMediaVault

```bash
ansible-playbook playbooks/patch_omv.yml
```

OMV is treated as **stateful storage** and is patched during a dedicated maintenance window.

---

### Ansible Control Node

```bash
ansible-playbook playbooks/patch_ansible.yml
```

The control node is patched last, and never while Semaphore jobs are running.

---

## Post-Patch Verification

After patching, I verify system state explicitly:

```bash
ansible TARGET -a "uptime -p"
ansible TARGET -a "who -b"
ansible TARGET -a "systemctl status qemu-guest-agent"
```

A host is not considered patched until these checks pass.

---

## Failure Handling Philosophy

Failures are surfaced immediately and handled conservatively.

- Partial success is acceptable if understood
- Silent failure is not acceptable
- Reboots are never retried blindly
- Recovery is manual unless explicitly automated

---

## Security Considerations

- SSH key-based authentication only
- Root SSH permitted only where operationally required
- No secrets hardcoded in playbooks
- Tier isolation limits blast radius

---

## What This Framework Does Not Do

- Modify networking or firewall configuration
- Upgrade applications beyond OS packages
- Patch clustered systems in parallel
- Automate recovery without verification

These boundaries are intentional.

---

## Summary

This patching framework reflects a deliberate balance between automation and control. Every decision favors **observability, safety, and repeatability** over speed or convenience.

If a system can be added to inventory, validated, patched, and verified using these steps, it belongs in the framework. If it cannot, it remains outside by design.

---
