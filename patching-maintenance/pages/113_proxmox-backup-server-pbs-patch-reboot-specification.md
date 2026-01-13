## Purpose

This document defines the **authoritative patching and reboot behavior** for the Proxmox Backup Server (PBS) in the cfolino.com infrastructure.

PBS is a **stateful backup appliance** responsible for protecting all virtual and physical workloads. Any disruption during active jobs can result in:
- Failed or corrupt backups
- Incomplete deduplication metadata
- Snapshot chain inconsistencies

For this reason, PBS patching is **strictly gated, externally rebooted, and fully verified**.

---

## Role & Playbook

- **Role:** `pbs_patch`
- **Playbook:** `playbooks/patch_pbs.yml`
- **Execution Scope:** Single host (`pbs`)
- **Automation Authority:** Ansible via Semaphore
- **Reboot Authority:** Proxmox VE (`qm reboot`)

PBS patching is never executed concurrently with:
- Backup jobs
- Hypervisor patching
- Storage maintenance

---

## Preconditions (Hard Gates)

The following checks are **mandatory**. Failure of any condition **aborts patching before package changes occur**.

### Proxmox Backup Tooling Validation
- `proxmox-backup-manager` must be present
- Confirms host is a valid PBS system

---

### Root Filesystem Free Space
- Command: `df -BG --output=avail /`
- Parsed numerically (GiB, not percentage)
- Must meet or exceed `pbs_min_root_free_gb`
- Prevents upgrades on space-constrained backup appliances

This check is **skipped only in `--check` mode**.

---

### Active Backup Task Detection
- Command: `proxmox-backup-manager task list`
- If any tasks are running, execution fails immediately
- Prevents interruption of active backups

This is a **hard stop** and requires operator intervention.

---

### APT / DPKG State Safety
PBS explicitly guards against package manager failure modes:
- Detects running `apt` / `dpkg` processes
- Waits (bounded) for active processes to exit
- Forces dpkg consistency via `dpkg --configure -a`
- Lock files are inspected but never blindly waited on

This prevents indefinite hangs and half-configured states.

---

## Patch Behavior

### Package Preparation
- Broken dependencies fixed via:
  - `apt-get --fix-broken install`
- Apt cache refreshed

---

### System Upgrade
- Full system upgrade using `dist-upgrade`
- Configuration files preserved (`confdef`, `confold`)
- Timeout enforced to prevent hung upgrades
- Exit code `124` (timeout) treated as non-fatal

This ensures unattended upgrades remain bounded and observable.

---

### Reboot Requirement Detection
- `/var/run/reboot-required` is checked
- Result is recorded for reporting
- **Reboot is always performed**, regardless of flag

PBS never remains running after patching.

---

## Reboot Semantics (Critical)

### Reboot Method
- Reboot is executed **from Proxmox VE** using:
  - `qm reboot <vmid>`
- PBS does not reboot itself

### Deterministic State Tracking
After issuing the reboot command:
1. PBS SSH is observed transitioning to `stopped`
2. PBS SSH is observed returning to `started`
3. System availability is explicitly confirmed

This guarantees a full power-cycle occurred.

---

### Guest Agent Integration
- If QEMU Guest Agent is active:
  - Uptime is retrieved post-reboot
- Absence of QGA does **not** fail execution

Guest Agent usage is opportunistic, not required.

---

## Post-Reboot Health Verification

After PBS returns online, the following checks are enforced:

### Service Validation
- All expected PBS services must be:
  - Running
  - Enabled
- Failure of any service fails the play

---

### Version & Task Sanity
- PBS package versions retrieved
- Last 10 PBS tasks listed for visibility
- Health summary fact constructed for reporting

This provides immediate post-maintenance confidence.

---

## Notifications

### Timing
- Notification is sent **after successful reboot and verification**
- Confirms system is operational, not merely restarted

---

### Contents
Notifications include:
- Kernel version
- Root filesystem free space
- Reboot-required flag state
- Service health results
- Recent PBS task activity
- Structured health summary data

PBS notifications are **confirmation-based**, not intent-based.

---

## Scheduling Constraints (Non-Negotiable)

PBS patching must follow these rules:

- Never scheduled during active backup windows
- Never scheduled concurrently with PVE patching
- Never scheduled concurrently with OMV maintenance
- Always scheduled in a dedicated infrastructure window
- Must complete before Ansible control node maintenance

PBS maintenance assumes hypervisor stability.

---

## Expected “Failure” Conditions (Not Errors)

The following conditions are **normal and expected**:

- SSH disconnect during reboot
- Temporary unavailability during shutdown/startup
- QEMU Guest Agent data unavailable

These do not indicate failure unless recovery does not occur.

---

## Post-Maintenance Guarantees

After successful execution:
- PBS has completed a full reboot
- All expected services are running
- No backup jobs were interrupted
- System is ready to resume scheduled backups

Any deviation from this state results in play failure.

---

## Design Rationale Summary

PBS patching is intentionally:
- Conservative
- Externally controlled
- Strongly gated
- Fully verified

This ensures backups remain trustworthy and automation remains predictable.

---

## Change Control Notes

Any modification to this role must preserve:
- Active task detection and hard fail behavior
- External reboot via Proxmox
- Deterministic shutdown/startup observation
- Post-reboot service validation
- Bounded package upgrade execution

Relaxing these constraints introduces unacceptable backup risk.
