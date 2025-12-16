---

This page documents the Ansible automation responsible for **patching, rebooting, and validating Proxmox virtual machines (VMs)** in the homelab.

Unlike traditional in-guest patching workflows, this automation deliberately **reboots VMs at the Proxmox hypervisor level** to guarantee clean shutdowns, predictable restarts, and deterministic post-patch validation.

---

## Purpose

This automation exists to:

- Patch Ubuntu-based VMs safely and consistently
- Enforce the presence and health of **QEMU Guest Agent**
- Reboot VMs from the **Proxmox host**, not from inside the guest
- Prevent double reboots and undefined VM states
- Validate SSH and QEMU availability after restart

This approach prioritizes **hypervisor authority over guest autonomy**.

---

## Scope

### Included
- Ubuntu VMs (`ubuntu_vms`)
- Infrastructure service VMs
- Pi-hole VMs (with health checks)
- Certificate Authority VM
- Grafana, BookStack, NPM, PBS VMs

### Explicitly Excluded
- Proxmox VE host itself
- OpenMediaVault bare metal
- UniFi devices

Each has its own dedicated automation.

---

## Inventory Requirements

Each VM **must define a VMID** and **Proxmox host** in inventory:

```yaml
pbs:
  ansible_host: 192.168.x.x
  vmid: 101
  proxmox_host: pve
```

These values are **mandatory** for hypervisor-level restarts.

---

## Entry Playbook

### Patch All Ubuntu VMs
```bash
ansible-playbook playbooks/patch_all.yml
```

This playbook orchestrates:
1. Guest agent enforcement
2. In-guest patching
3. VM shutdown and restart via Proxmox
4. Post-reboot validation

---

## Execution Flow

### 1. Enforce QEMU Guest Agent

**Role:** `qemu_ga_fix`

Actions:
- Installs `qemu-guest-agent`
- Injects a systemd shutdown hook
- Ensures agent is running
- Waits for virtio socket presence

```text
/dev/virtio-ports/org.qemu.guest_agent.0
```

This guarantees Proxmox can:
- Shut down the VM cleanly
- Track VM lifecycle state
- Avoid forced power-offs

---

### 2. Apply OS Patches (In-Guest)

**Role:** `patch_reboot`

Actions:
- Updates apt cache
- Applies `dist-upgrade`
- Removes obsolete packages
- Ensures QEMU agent is running before reboot

Reboots are **not performed inside the VM**.

---

### 3. Hypervisor-Level VM Restart

**Role:** `proxmox_vm_restart`

Reboot sequence:

1. Assert `vmid` is defined
2. Issue shutdown from Proxmox:
   ```bash
   qm stop <vmid> --skiplock
   ```
3. Pause briefly for cleanup
4. Start VM:
   ```bash
   qm start <vmid>
   ```
5. Wait for:
   - SSH availability
   - QEMU virtio socket
6. Verify guest agent health

All Proxmox commands are executed via:
```yaml
delegate_to: "{{ proxmox_host }}"
```

---

## Why Hypervisor Reboots?

| Method | Reason Rejected |
|------|---------------|
| `reboot` inside VM | Double reboot risk |
| `shutdown -r now` | Guest-controlled |
| ACPI only | No verification |
| Proxmox API | CLI preferred for determinism |

Hypervisor control ensures:
- Single reboot
- Clean shutdown
- Predictable state recovery

---

## Pi-hole Health Integration

After VM restart:

```yaml
include_role:
  name: pihole_health
```

Only runs on:
- `pihole`
- `pihole-backup`

Confirms DNS service health before continuing.

---

## Failure Behavior

| Failure | Result |
|------|------|
| Missing VMID | Immediate abort |
| Guest agent missing | Auto-installed |
| SSH does not return | Play fails |
| QEMU socket missing | Play fails |
| Partial VM success | No cascading retries |

Each VM is treated independently.

---

## Safety Characteristics

- No forced reboots
- No parallel VM shutdowns
- No Proxmox host changes
- Explicit inventory bindings
- No implicit discovery

---

## Validation Commands

### List Tasks (Dry Inspection)
```bash
ansible-playbook playbooks/patch_all.yml --list-tasks
```

### Confirm VM State on Proxmox
```bash
qm status <vmid>
```

### Verify Guest Agent
```bash
systemctl status qemu-guest-agent
```

---

## Role Structure

```text
roles/
├── qemu_ga_fix/
├── patch_reboot/
└── proxmox_vm_restart/
```

Each role represents a **single responsibility**:
- Agent health
- OS patching
- Hypervisor lifecycle

---

## Design Principles

- Proxmox is the authority
- Guests do not reboot themselves
- Guest agent is mandatory
- One reboot per cycle
- Visibility before completion

---

## Related Pages

- Proxmox VE Host Patching
- OpenMediaVault Automation
- Monitoring & Exporters
- Patch Orchestration Strategy

---
