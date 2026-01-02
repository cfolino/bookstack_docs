---
## Purpose

This page explains how the **QEMU Guest Agent (QGA)** is used within the environment and why it is considered an important dependency for **safe automation and recovery**, while deliberately **not acting as a lifecycle authority**.

QGA is treated as a **coordination and communication mechanism** between the guest and the hypervisor, not as a control mechanism for guest-initiated power actions.

---

## What QGA Provides

Within this environment, QGA serves as:

- A communication channel between Proxmox and the guest OS
- A mechanism for graceful shutdowns when initiated by the hypervisor
- A source of guest introspection and state reporting
- A safety requirement for hypervisor-driven recovery operations

QGA is intentionally **not** used to make decisions about guest health, readiness, or reboot success.

---

## What QGA Does Not Control

QGA is not used as:

- A guest reboot trigger
- A signal that system initialization has fully completed
- A replacement for SSH-based verification
- A general-purpose automation gate

The guest operating system remains responsible for its own lifecycle decisions.

---

## Why QGA Is Still Required

From a hypervisor perspective, a VM without QGA:

- Is harder to shut down cleanly from Proxmox
- Is more difficult to recover if it becomes unresponsive
- May require forced power-off actions during maintenance

While a VM can function without QGA, recovery operations are more predictable and safer when the agent is present and healthy.

---

## Role of QGA in This Environment

QGA is used for:

- Graceful shutdowns initiated by Proxmox
- Hypervisor-side lifecycle actions during recovery
- Guest state inspection
- Coordination during exceptional situations

QGA is not used for:

- Guest-initiated reboots
- Patch success validation
- Determining system readiness after boot
- Blocking or allowing automation progress

---

## Enforcement Strategy

QGA presence and health are verified before any **hypervisor-level lifecycle action** is attempted.

This verification is handled by:

```
roles/qemu_ga_fix
```

The role ensures that Proxmox can safely interact with the guest if recovery becomes necessary.

---

## What the `qemu_ga_fix` Role Ensures

The role maintains the following conditions:

1. The QEMU Guest Agent package is installed
2. The agent service is enabled and running
3. No guest-side shutdown hooks are configured
4. Systemd state is consistent
5. The virtio communication socket exists
6. The agent responds to Proxmox queries

If these conditions are not met, automation avoids hypervisor-driven lifecycle actions.

---

## Guest-Initiated Shutdown Hooks

Earlier designs experimented with guest-side shutdown hooks tied to the QGA service lifecycle. In practice, these hooks introduced unintended behavior, including shutdowns triggered during service restarts or package upgrades.

For this reason, the environment avoids configurations such as:

```
ExecStopPost=/usr/bin/qemu-ga --shutdown
```

The guest OS does not request shutdown through QGA.
Shutdown requests originate from Proxmox when required.

---

## Readiness and Health Verification

### Guest Readiness

Guest health and readiness are determined using:

- SSH reachability
- Successful command execution
- Guest OS uptime

These signals are sufficient to determine whether the system is operational for automation purposes.

### Hypervisor Readiness

QGA socket availability confirms that:

- Proxmox can communicate with the guest
- Hypervisor-initiated actions can be performed safely if needed

These two concerns are intentionally separated.

---

## Why SSH Remains the Primary Signal

SSH confirms that:

- The network stack is functional
- User-space services are operational
- The system can accept automation tasks

Systemd state and QGA availability are supplementary indicators and are not treated as authoritative signals of guest readiness.

---

## Interaction With VM Lifecycle Control

QGA is required for:

- Hypervisor-initiated shutdowns
- Recovery workflows such as `proxmox_vm_restart`
- Avoiding forced power-off operations

QGA is not required for:

- Guest-initiated reboots
- Routine patching
- Automation flow control

---

## Failure Handling

If QGA is unavailable or unhealthy:

- Guest patching may still proceed
- Hypervisor recovery actions are avoided
- Automation fails early only when hypervisor control would otherwise be required

This approach favors predictable failure modes over forced intervention.

---

## Design Rationale

Separating responsibilities yields clearer behavior:

- Guest OS controls its own lifecycle
- Proxmox controls recovery actions
- QGA enables coordination without dictating intent

This boundary reduces unintended side effects and simplifies troubleshooting.

---

## Summary

- QEMU Guest Agent is required for safe hypervisor interaction
- QGA does not act as a reboot authority
- Guest OS decisions are verified via SSH
- Hypervisor recovery depends on QGA availability
- Guest-side shutdown hooks are intentionally avoided

This model balances safety, clarity, and operational predictability.

---

## Related Pages

- VM Lifecycle & Reboot Semantics
- Proxmox VE Host Patching
- Bulk Ubuntu VM Patching
- Global Patching Orchestration Model
---
