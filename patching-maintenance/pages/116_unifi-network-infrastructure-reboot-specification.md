## Purpose

This document defines the **authoritative reboot behavior** for UniFi network infrastructure in the cfolino.com environment.

UniFi devices provide:
- Network connectivity
- Inter-VLAN routing (via UDM Pro)
- Wireless access
- Management and telemetry for switching and APs

Improper reboot sequencing can result in:
- Loss of controller access
- Incomplete device reboots
- API failures
- Extended network outages

For this reason, UniFi reboots are **explicitly segmented by device type** and executed with strict ordering and verification.

---

## Role & Playbooks

- **Role:** `unifi_reboot`
- **Playbooks:**
  - UDM Pro reboot (SSH-based)
  - Non-UDM UniFi device reboot (API-based)
- **Execution Scope:** `localhost`
- **Automation Authority:** Ansible via Semaphore
- **Reboot Authority:** SSH (UDM) and UniFi Network API (devices)

UniFi reboots are never combined with other infrastructure maintenance.

---

## Architectural Separation (Critical)

UniFi infrastructure is rebooted in **two distinct phases**:

1. **UDM Pro (Gateway + Controller)**
2. **All other UniFi devices (switches, access points)**

This separation is **non-negotiable**.

| Component | Reboot Method | Reason |
|---|---|---|
| **UDM Pro** | SSH reboot | API unavailable during reboot |
| **Switches / APs** | UniFi API | Scalable, state-aware |

Attempting to reboot all devices uniformly is unsafe and unsupported.

---

## UDM Pro Reboot Specification

### Reboot Method
- Executed via SSH:
  ```text
  sudo /sbin/reboot
