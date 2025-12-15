---

This page documents the **operational verification procedures** used before and after patching workflows. These checks serve as the final authority on whether a maintenance operation is considered successful.

Automation executes changes; verification confirms outcomes. Both are required.

---

## Purpose

Operational runbooks exist to:

- Establish baseline system state before patching
- Confirm that patching and reboots actually occurred
- Detect silent or partial failures
- Provide a repeatable verification process
- Define clear stop conditions

A system is not considered “patched” until verification completes successfully.

---

## Pre-Patch Verification Checklist

Before executing any patching workflow, baseline state is captured.

### Connectivity

```bash
ansible TARGET -m ping
```

Confirms:
- Inventory correctness
- SSH connectivity
- Basic reachability

---

### Disk Space

```bash
ansible TARGET -a "df -h /"
```

Confirms:
- Adequate space for package upgrades
- No immediate disk pressure

---

### Uptime Baseline

```bash
ansible TARGET -a "uptime -p"
ansible TARGET -a "who -b"
```

Records:
- How long the system has been running
- Last boot timestamp

This establishes a reference point for post-patch validation.

---

### VM-Specific Check (If Applicable)

```bash
ansible TARGET -a "systemctl status qemu-guest-agent"
```

Confirms:
- Guest agent is running
- Reboot visibility will be reliable

If this check fails, patching does not proceed.

---

## Patch Execution Reference

Execution commands are documented per workflow:

```bash
ansible-playbook playbooks/patch_all.yml
ansible-playbook playbooks/patch_pve.yml
ansible-playbook playbooks/patch_pbs.yml
ansible-playbook playbooks/patch_omv.yml
ansible-playbook playbooks/patch_ansible.yml
```

Dry run validation:

```bash
ansible-playbook playbooks/<playbook>.yml --check
```

Dry runs validate structure and scope but do not guarantee runtime success.

---

## Post-Patch Verification Checklist

After patching completes, verification is mandatory.

---

### Uptime Confirmation

```bash
ansible TARGET -a "uptime -p"
ansible TARGET -a "who -b"
```

Expected outcomes:
- Uptime reflects recent reboot (if required)
- Boot timestamp falls within maintenance window

If no reboot was required, uptime should be unchanged.

---

### Service-Level Verification (VMs)

```bash
ansible TARGET -a "systemctl is-active qemu-guest-agent"
```

Expected output:

```text
active
```

Confirms:
- Guest agent survived reboot
- Proxmox state tracking is restored

---

### Infrastructure-Specific Checks

#### Proxmox VE

```bash
ssh root@<internal-host> pveversion
ssh root@<internal-host> zpool status
```

Expected:
- Version reflects updates
- ZFS pools healthy

---

#### Proxmox Backup Server

```bash
ssh root@<internal-host> proxmox-backup-manager version
ssh root@<internal-host> proxmox-backup-manager datastore list
```

Expected:
- PBS version updated
- Datastores accessible

---

#### OpenMediaVault

```bash
ssh root@<internal-host> omv-version
ssh root@<internal-host> systemctl status omv-engined
```

Expected:
- OMV services active
- No filesystem errors

---

## Application-Level Verification

Where applicable, service health checks are executed via Ansible roles:

- DNS resolution (Pi-hole)
- HTTP endpoints (Grafana, BookStack)
- Service state assertions

Failures at this stage indicate **application-level issues**, not patch failures.

---

## Failure Classification

Failures are categorized to guide response:

| Category | Description |
|-------|------------|
| Pre-flight | Environment not ready |
| Patch | Package or dependency failure |
| Reboot | Host did not return |
| Service | Application did not recover |
| Visibility | Notification or logging failure |

Each category has a different recovery approach.

---

## Stop Conditions

Automation and verification **must stop** when:

- SSH connectivity does not return
- ZFS pools are degraded
- Datastores are unavailable
- Critical services fail health checks
- Results cannot be confidently validated

Proceeding without resolution is considered unsafe.

---

## Recovery Philosophy

Recovery is:

- Manual by default
- Targeted to the affected system
- Performed before resuming automation
- Verified independently before retry

Automation is never used to blindly “fix” failures.

---

## Documentation & Traceability

For each maintenance window:

- Execution logs are retained
- Notifications summarize outcomes
- Verification results are observable
- Failures are explainable

If outcomes cannot be reconstructed, the process is considered incomplete.

---

## Summary

Operational verification is the **final gate** in the patching framework. It ensures that automation results in real, observable system health — not just successful playbook execution.

A patch run that cannot be verified is treated as a failed run, regardless of automation output.

---
