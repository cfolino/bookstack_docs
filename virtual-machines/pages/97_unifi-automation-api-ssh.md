---

This page documents the Ansible automation used to interact with and manage UniFi networking infrastructure in the homelab. The automation focuses on **safe visibility**, **controlled reboots**, and **health verification** of UniFi devices, while respecting the operational differences between the UDM and downstream network devices.

UniFi automation is treated as **infrastructure-adjacent**, not generic Linux automation.

---

## Purpose

UniFi automation exists to:

- Verify UniFi device reachability and uptime
- Perform controlled reboots of UniFi devices
- Distinguish between UDM and non-UDM devices
- Avoid unsafe or blind network disruptions
- Provide visibility into network state before and after changes

All automation is designed to be **intentional, explicit, and reversible**.

---

## Managed Devices

### UniFi Dream Machine (UDM / UDM Pro)
- Acts as:
  - Gateway
  - UniFi Controller
  - Network authority
- Requires **special handling**
- Reboots are disruptive and treated separately

### UniFi Network Devices
Includes:
- Switches
- Access points

These devices are managed primarily via the **UniFi Controller API**.

---

## Inventory Group

```yaml
unifi:
  hosts:
    udmpro:
```

UniFi automation targets are intentionally isolated into a dedicated group to prevent accidental inclusion in general patching workflows.

---

## Entry Playbooks

| Playbook | Purpose |
|--------|--------|
| `unifi_preflight_check.yml` | Validate controller connectivity |
| `unifi_uptime_check.yml` | Collect uptime data |
| `unifi_reboot_devices.yml` | Reboot switches/APs via API |
| `unifi_reboot_udm.yml` | Reboot UDM via SSH |
| `unifi_debug_*` | Diagnostics and API validation |

---

## Preflight Checks

### Purpose
Preflight checks validate that:
- UniFi Controller is reachable
- API authentication works
- Target devices are visible

This prevents blind execution against an unreachable controller.

### Playbook
```bash
ansible-playbook playbooks/unifi_preflight_check.yml
```

Failures here **abort all further UniFi automation**.

---

## Uptime Verification

### Playbook
```bash
ansible-playbook playbooks/unifi_uptime_check.yml
```

This playbook:
- Queries the UniFi API
- Retrieves device uptime
- Confirms controller visibility

Used to:
- Verify network stability
- Confirm reboot success
- Detect unexpected resets

---

## Rebooting Non-UDM Devices (API)

### Playbook
```bash
ansible-playbook playbooks/unifi_reboot_devices.yml
```

### Behavior
- Uses UniFi Controller API
- Reboots switches/APs individually
- Collects results per device
- Generates a summary report

### Why API Reboots
- Graceful device handling
- No SSH access required
- Controller-aware orchestration
- Reduced risk of orphaned devices

---

## Rebooting the UDM (SSH)

### Playbook
```bash
ansible-playbook playbooks/unifi_reboot_udm.yml
```

### Why SSH Is Required
- The UDM hosts the controller itself
- API-based reboot would sever its own control plane
- SSH provides a deterministic reboot path

### Safety Characteristics
- Explicit host targeting
- No bulk execution
- No retries
- Manual intent required

UDM reboots are **never chained** with other UniFi actions.

---

## Role Structure

```text
roles/unifi_reboot/
├── tasks/
│   ├── main.yml
│   ├── reboot_api.yml
│   ├── reboot_api_per_device.yml
│   ├── reboot_device.yml
│   ├── reboot_udm.yml
│   └── summary.yml
```

Each task file represents a **distinct operational boundary**.

---

## Delegation & Execution Model

- API calls execute from the Ansible control node
- SSH operations explicitly target the UDM
- No UniFi device executes Ansible locally
- No credentials are distributed to network devices

This keeps UniFi automation **control-plane only**.

---

## Verification & Validation

### Confirm Device Status
```bash
ansible-playbook playbooks/unifi_uptime_check.yml
```

### UniFi Controller UI
- Devices → Status
- Confirm uptime reset where expected
- Confirm all devices return to “Connected”

---

## Failure Behavior

| Scenario | Result |
|--------|-------|
| API unreachable | Playbook fails early |
| Device reboot fails | Reported in summary |
| UDM reboot fails | Manual recovery required |
| Partial success | No retries or cascades |

No automatic retries are performed on network infrastructure.

---

## Security Model

- API credentials stored securely
- No SSH access to switches/APs
- SSH access restricted to UDM only
- No plaintext secrets in playbooks
- Explicit targeting only

---

## Design Principles

- Network stability over automation speed
- Explicit intent for disruptive actions
- Separate UDM from downstream devices
- Visibility before and after changes
- No hidden behavior

---

## Maintenance Notes

### Add New UniFi Device
- Adopt device in UniFi Controller
- No inventory change required
- Device becomes visible automatically

### Remove Device
- Remove from controller
- Automation adapts dynamically

---

## Related Pages

- Ansible Inventory & Group Design
- Patch Orchestration Strategy
- Monitoring & Observability
- Network Architecture Overview

---
