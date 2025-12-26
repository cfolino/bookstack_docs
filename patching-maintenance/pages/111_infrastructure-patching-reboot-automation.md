## Overview

This page documents the infrastructure patching and reboot automation implemented across the cfolino.com homelab. These automations are designed to support **unattended, low-risk maintenance** while preserving service availability, observability, and recovery guarantees.

All patching workflows are:
- Host-scoped (one system per run)
- Role-driven
- Notification-backed
- Designed for scheduled execution via Semaphore

Infrastructure systems are intentionally handled **differently**, based on their role in the environment.

---

## Design Philosophy

Infrastructure patching follows these principles:

- One Ansible role per system class
- Explicit pre-flight safety checks
- Reboots are intentional and deterministic
- Control-plane disruptions are planned
- Redundant services are never rebooted concurrently
- Notifications are sent before loss of connectivity
- Scheduling enforces blast-radius containment

This is not “generic patching” — each system is treated according to its operational risk.

---

## Inventory Grouping

Infrastructure hosts are grouped as follows:

- `infrastructure`
  - `pve`
  - `pbs`
  - `omv`
  - `ansible`
- `dns`
  - `pihole`
  - `pihole-backup`
- `unifi`
  - `udmpro`

Semaphore jobs always target **a single host**, never a group.

---

## Proxmox VE (PVE)

**Role:** `pve_patch`
**Nature:** Hypervisor / infrastructure control plane
**Reboot Method:** Local asynchronous reboot

### Key Characteristics
- Performs ZFS health checks
- Enforces root filesystem free-space thresholds
- Uses `dist-upgrade`
- Sends notification **before reboot**
- Reboots asynchronously (`/sbin/reboot`)
- SSH disconnect is expected and treated as success

### Important Behavior
Rebooting PVE will temporarily interrupt hosted VMs, including the Ansible control node. This is intentional and must be scheduled in isolation.

---

## Proxmox Backup Server (PBS)

**Role:** `pbs_patch`
**Nature:** Backup appliance VM
**Reboot Method:** `qm reboot` via Proxmox host

### Key Characteristics
- Hard fails if backup jobs are running
- Enforces numeric free-space thresholds
- Guards against dpkg/apt lock issues
- Applies `dist-upgrade` with timeout protection
- Reboot is orchestrated from the hypervisor
- Waits for shutdown and startup deterministically
- Performs post-reboot service validation

### Design Rationale
PBS never reboots itself. All reboots are controlled externally to prevent half-patched states and ensure predictable recovery.

---

## OpenMediaVault (OMV)

**Role:** `omv_patch`
**Nature:** Bare-metal storage host
**Reboot Method:** Conditional local reboot

### Key Characteristics
- Reboots only if required
- Gracefully stops Samba services before reboot
- Captures pre/post uptime and network reachability
- Optional Btrfs health checks
- Builds a rolling patch summary
- Designed to minimize client disruption

OMV is treated as a storage appliance, not a general-purpose server.

---

## UniFi Infrastructure

**Role:** `unifi_reboot`
**Nature:** Network management and switching
**Reboot Authority:** Split by device type

### UDM Pro
- Rebooted via SSH
- API is unavailable during reboot
- Offline/online state is explicitly verified
- Treated as a control-plane event

### Switches / Access Points
- Rebooted via UniFi Network API
- Gateways explicitly excluded
- Only connected devices are targeted
- API calls are throttled
- Summary report generated after controller recovery

This separation avoids the most common UniFi automation failure modes.

---

## Ansible Control Node

**Role:** `ansible_node_patch`
**Nature:** Automation control plane
**Reboot Method:** `qm reboot` via Proxmox

### Key Characteristics
- Captures extensive pre/post system state
- Optionally pulls automation repository from GitHub
- Hard fails on uncommitted local changes
- Backs up critical files before patching
- Stops Semaphore cleanly before reboot
- Reboots via hypervisor
- Verifies Semaphore service post-reboot

This role is always executed **last**.

---

## Notifications

All infrastructure patching workflows generate HTML email summaries including:
- System identity
- Patch activity
- Reboot behavior
- Health verification results

Notifications are sent **before** connectivity loss whenever applicable.

---

## Scheduling Rules (Semaphore)

The following rules are enforced operationally:

- One Semaphore job per host
- No concurrent infrastructure reboots
- PVE scheduled in its own window
- PBS never scheduled during active backups
- Primary and backup DNS never patched together
- Ansible control node scheduled last
- Retries disabled; failures require manual review

Scheduling is a control mechanism, not a convenience feature.

---

## Adding a New Infrastructure Host

To add a new infrastructure system:

1. Create a dedicated role with explicit safety checks
2. Ensure reboot behavior is deterministic
3. Integrate notification via `notify_email`
4. Add the host to inventory
5. Create a dedicated Semaphore job
6. Document reboot semantics and scheduling constraints

Automation is extended deliberately, not opportunistically.

---

## Closing Notes

This patching framework is designed to be:
- Predictable
- Observable
- Recoverable
- Portfolio-grade

Every deviation from “default Ansible behavior” is intentional and documented.
