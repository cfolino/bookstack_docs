## Purpose

This document defines the **authoritative patching and reboot behavior** for the Ansible control node in the cfolino.com infrastructure.

The Ansible node is the **automation control plane** responsible for:
- Executing all infrastructure patching
- Hosting the Semaphore scheduler
- Maintaining the authoritative automation repository

Maintenance on this host must preserve **automation integrity, repository consistency, and recoverability**. For this reason, Ansible node patching is intentionally **last, isolated, and externally rebooted**.

---

## Role & Playbook

- **Role:** `ansible_node_patch`
- **Playbook:** `playbooks/patch_ansible_node.yml`
- **Execution Scope:** Single host (`ansible`)
- **Automation Authority:** Semaphore (external scheduler)
- **Reboot Authority:** Proxmox VE (`qm reboot`)

The Ansible node never schedules or reboots itself.

---

## Scheduling Position (Hard Rule)

The Ansible control node **must always be patched last**.

It is never scheduled:
- Concurrently with any other infrastructure job
- During hypervisor maintenance
- During DNS maintenance
- During UniFi control-plane changes

All upstream infrastructure must already be stable.

---

## Preconditions (Hard Gates)

The following checks must pass before patching proceeds.

### Repository State Validation
- Local automation repository status is captured
- If `ansible_node_git_pull=true`:
  - Repository is updated from GitHub
  - Execution fails if uncommitted local changes exist

This prevents silent loss of in-flight automation work.

---

### Baseline System Capture
Before any changes:
- OS version
- Kernel version
- Uptime
- Root filesystem usage
- Semaphore service status
- List of upgradable packages

This data forms the **pre-maintenance audit baseline**.

---

## Backup Guarantees (Critical)

Before patching:
- A timestamped backup archive is created containing:
  - Ansible SSH keys
  - Semaphore systemd service files
  - Semaphore configuration data
- Backup directory permissions are enforced
- Archive ownership is corrected post-creation

This guarantees rapid recovery in the event of failure.

---

## Patch Behavior

### Package Management
- Apt cache refreshed
- Full system upgrade performed using:
  - `apt dist-upgrade`
  - `autoremove` and `autoclean` enabled

### Change Visibility
- List of upgradable packages is captured **before and after**
- Kernel version change is explicitly detected and recorded

---

## Semaphore Service Handling

### Pre-Reboot Handling
- Semaphore service is stopped cleanly
- Post-stop status is recorded

This prevents:
- Mid-job termination
- Database or state corruption
- Inconsistent scheduler state

---

## Reboot Semantics (Critical)

### Reboot Method
- Reboot is executed externally via:
  - `qm reboot <vmid>` on Proxmox VE
- The Ansible node never reboots itself

### Expected Behavior
- SSH connectivity is lost
- Semaphore is offline during reboot
- No callbacks are expected during reboot

This is **intentional and correct**.

---

## Post-Reboot Verification

After the node returns online:

### Connectivity
- SSH availability is explicitly waited for

### Service Validation
- Semaphore service is started
- Service enablement is enforced
- Active state is asserted
- Failure to start Semaphore fails the play

Automation authority is not considered restored until Semaphore is confirmed active.

---

## Notification Behavior

### Timing
- Notification is sent **before reboot**
- Confirms patch actions and reboot intent

### Contents
Notifications include:
- OS and kernel before/after
- Uptime before patching
- Disk usage before/after
- Git repository status
- Semaphore service state
- Apt upgrade results
- Backup archive path
- Reboot execution note

Notifications serve as the **authoritative audit record**.

---

## Expected “Failure” Conditions (Not Errors)

The following are **expected** and do not indicate failure:

- Loss of SSH connectivity during reboot
- Semaphore unavailable during reboot window
- Temporary scheduling blackout

These outcomes are intrinsic to control-plane maintenance.

---

## Post-Maintenance Guarantees

After successful execution:
- System is fully patched
- Kernel changes are recorded
- Semaphore service is active and enabled
- Automation repository is in a known-good state
- Backup artifacts exist for rollback

Only after these guarantees are met is the automation plane considered restored.

---

## Design Rationale Summary

The Ansible control node patching process is intentionally:
- Conservative
- State-aware
- Externally controlled
- Recovery-focused

This ensures automation remains trustworthy and failures are diagnosable.

---

## Change Control Notes

Any modification to this role must preserve:
- Repository cleanliness enforcement
- Pre-patch backups
- External reboot via Proxmox
- Explicit Semaphore shutdown/startup
- Post-reboot service assertions

Relaxing these constraints risks automation integrity and system recoverability.
