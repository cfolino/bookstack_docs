---

This page documents why the **QEMU Guest Agent (QGA)** is treated as a **mandatory dependency** for automation and VM lifecycle orchestration.

QGA is not an optimization.
It is a **control plane requirement**.

---

## Why the QEMU Guest Agent Matters

From the perspective of automation, a virtual machine without QGA is:

- Operationally opaque
- Unsafe to reboot unattended
- Unable to reliably signal readiness
- Prone to partial shutdown states

While a VM *can* run without QGA, **automation cannot safely manage it**.

---

## Role of QGA in This Environment

QGA is relied upon for:

- Clean shutdown signaling
- Accurate reboot completion
- Guest readiness verification
- Hypervisor ↔ guest coordination

Without QGA, Proxmox cannot reliably:
- Know when a VM has actually shut down
- Know when the guest OS is ready after boot
- Perform safe lifecycle transitions during patching

---

## Enforcement Strategy

QGA is not assumed to be present.

It is **explicitly enforced** by automation before any lifecycle action occurs.

This enforcement is handled by the role:

```
roles/qemu_ga_fix
```

---

## What the `qemu_ga_fix` Role Guarantees

The role ensures the following invariants:

1. **QEMU Guest Agent package is installed**
2. **Systemd service is enabled and running**
3. **Shutdown hook is present**
4. **Systemd daemon state is correct**
5. **Virtio socket exists**
6. **Agent responds before lifecycle actions proceed**

If any of these checks fail, automation **halts immediately**.

---

## Shutdown Hook Enforcement

A systemd drop-in is installed:

```
/etc/systemd/system/qemu-guest-agent.service.d/notify.conf
```

Purpose:
- Ensure Proxmox receives a shutdown notification
- Prevent forced power-off scenarios
- Avoid filesystem inconsistency during reboots

This is critical during patch-driven restarts.

---

## Readiness Verification

After restarts, automation waits for:

```
/dev/virtio-ports/org.qemu.guest_agent.0
```

This socket confirms:
- The guest OS is fully booted
- The agent is active
- Proxmox ↔ guest communication is established

SSH availability alone is **not sufficient**.

---

## Why SSH Is Not Enough

SSH only proves:
- The network stack is up
- A daemon is listening

It does **not** guarantee:
- System services are stable
- Init has completed
- Shutdown signaling will work

QGA provides **hypervisor-level confirmation**, which SSH cannot.

---

## Interaction With VM Lifecycle Control

QGA is a prerequisite for:

- `proxmox_vm_restart`
- `patch_all.yml`
- `patch_ansible_node.yml`
- `patch_pbs.yml`

Lifecycle actions **do not proceed** unless QGA is confirmed healthy.

This prevents:
- Half-rebooted VMs
- Hung shutdowns
- Orphaned processes
- Proxmox task deadlocks

---

## Failure Behavior (Intentional)

If QGA is missing or broken:

- Automation fails fast
- No reboot is attempted
- No patching continues
- Operator intervention is required

This is intentional and preferable to undefined state.

---

## Why This Is a Hard Requirement

Treating QGA as optional leads to:

- Unreliable unattended patching
- Non-deterministic VM state
- Increased recovery time
- Higher risk of data corruption

Treating QGA as mandatory yields:

- Predictable lifecycle control
- Safe delegation
- Clean orchestration boundaries

---

## Summary

- QEMU Guest Agent is a **control dependency**
- It is enforced, not assumed
- Lifecycle automation depends on it
- SSH alone is insufficient
- Fail-fast behavior is intentional

Any VM without a healthy QGA instance is **not eligible for automation**.

---

## Related Pages

- Proxmox-Driven VM Lifecycle Control
- Patch & Reboot Orchestration Strategy
- Inventory & Grouping Strategy
