---

This page documents how **virtual machine lifecycle actions** (shutdown, reboot, restart) are intentionally **orchestrated from the Proxmox host**, not from inside guest VMs.

This design is foundational to safe automation across the homelab and is used by **all patching and reboot workflows**.

---

## Design Principle

**The hypervisor is the source of truth for VM state.**

Guest operating systems:
- Cannot reliably determine their own power state
- May be degraded during patching
- Cannot guarantee clean shutdown semantics

Proxmox:
- Always knows VM state
- Controls QEMU lifecycle directly
- Can enforce shutdown ordering
- Remains reachable even when guests are unstable

For these reasons, **VM lifecycle actions are never executed from within the VM itself**.

---

## Control Flow Overview

The lifecycle model follows a strict pattern:

1. **Playbook targets the VM**
2. **VM metadata (VMID) is read from inventory**
3. **Lifecycle action is delegated to Proxmox**
4. **Guest availability is verified post-action**

This separation ensures:
- No circular dependencies
- No reliance on guest services during shutdown
- Predictable recovery behavior

---

## Inventory as Lifecycle Metadata

Each VM defines its Proxmox identity in `host_vars`.

### Example

```
inventory/host_vars/pbs.yml
```

```
vmid: 101
```

The `vmid` variable is:
- Not a service attribute
- Not an OS attribute
- A **hypervisor orchestration identifier**

This allows generic roles to operate on *any* VM without hardcoding.

---

## Core Roles Involved

### `proxmox_vm_restart`

This role performs **full VM stop/start cycles** using Proxmox CLI commands.

Key behaviors:
- Validates that `vmid` is defined
- Issues `qm stop` from the Proxmox host
- Waits for QEMU cleanup
- Issues `qm start`
- Verifies guest SSH availability
- Confirms QEMU Guest Agent readiness

### `qemu_ga_fix`

This role enforces **QEMU Guest Agent presence and correctness** before lifecycle actions.

It ensures:
- Agent is installed
- Shutdown hooks exist
- Systemd is correctly configured
- Virtio socket is present

Lifecycle actions never proceed unless GA is healthy.

---

## Delegation Mechanics

Lifecycle commands are executed using **Ansible delegation**.

### Example

```
delegate_to: pve
command: qm reboot {{ vmid }}
```

Important characteristics:
- Commands run *on the Proxmox host*
- SSH connectivity to the VM is irrelevant at this stage
- Failure of the guest does not block control flow

Delegation isolates **control authority** from **execution context**.

---

## Why `qm reboot` / `qm stop` Is Preferred

Using Proxmox commands instead of in-guest reboots ensures:

- Hypervisor-level visibility
- Accurate state transitions
- No dependency on init systems
- Clean power control even during OS failure

Guest-initiated reboots are intentionally avoided for automation.

---

## Post-Reboot Verification

After lifecycle actions:

- SSH availability is verified
- QEMU Guest Agent socket is confirmed
- Services are validated by downstream roles

This guarantees:
- The VM is actually online
- It is manageable
- Subsequent automation can proceed safely

---

## Where This Is Used

This lifecycle model is used by:

- `patch_all.yml` (Ubuntu VMs)
- `patch_ansible_node.yml`
- `patch_pbs.yml`
- Any playbook requiring controlled reboots

All VM restarts follow the same control path.

---

## Failure Domains & Safety

This design ensures:

- Proxmox failures do not cascade into guest logic
- Guest failures do not block hypervisor control
- Reboots remain deterministic
- Automation fails fast when metadata is missing

If `vmid` is not defined, the playbook **intentionally fails**.

---

## Why This Matters

Treating Proxmox as the lifecycle authority:

- Prevents partial reboots
- Avoids ghost VM states
- Enables safe unattended patching
- Keeps automation resilient under stress

This is a **non-negotiable invariant** of the homelab.

---

## Summary

- VM lifecycle actions are hypervisor-controlled
- Guests never reboot themselves in automation
- Inventory defines orchestration identity
- QEMU Guest Agent is mandatory
- Delegation enforces clean separation of control

If this model is followed, **VM patching is safe, repeatable, and observable**.

---

## Related Pages

- Patch & Reboot Orchestration Strategy
- QEMU Guest Agent as a Control Dependency
- Proxmox VE Patching
- Inventory & Grouping Strategy
