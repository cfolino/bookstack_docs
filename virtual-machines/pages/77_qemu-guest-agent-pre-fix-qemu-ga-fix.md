---

This page documents the `qemu_ga_fix` role, which exists to **guarantee reliable reboot behavior for virtual machines running under Proxmox**. This role is a mandatory prerequisite for any patching workflow that may trigger a VM reboot.

Without a functioning QEMU Guest Agent, reboot automation is unreliable and monitoring systems may falsely report failures.

---

## Role Purpose

The purpose of the `qemu_ga_fix` role is to:

- Ensure the QEMU Guest Agent package is installed
- Ensure the service is enabled and running
- Guarantee Proxmox can accurately track VM power state
- Prevent false-positive failures during reboots
- Stabilize Semaphore and Ansible job reporting

This role does **not** perform patching or rebooting. It only prepares the VM to safely participate in those actions.

---

## Role Location

```text
roles/qemu_ga_fix/
├── tasks/
│   └── main.yml
```

This role is intentionally minimal and self-contained.

---

## Task Breakdown

### Task File

```yaml
# roles/qemu_ga_fix/tasks/main.yml

- name: Install QEMU Guest Agent
  ansible.builtin.apt:
    name: qemu-guest-agent
    state: present
  become: true

- name: Enable and start QEMU Guest Agent
  ansible.builtin.systemd:
    name: qemu-guest-agent
    state: started
    enabled: true
  become: true
```

---

## Task Explanation

### Install QEMU Guest Agent

```yaml
ansible.builtin.apt:
  name: qemu-guest-agent
  state: present
```

This task ensures the guest agent package is installed.

Key points:
- Safe to re-run
- No changes made if already installed
- Uses standard OS package management

---

### Enable and Start Service

```yaml
ansible.builtin.systemd:
  name: qemu-guest-agent
  state: started
  enabled: true
```

This task guarantees:
- The service is running now
- The service will start on future boots

This is critical for reboot visibility and lifecycle management.

---

## Why This Role Exists Separately

This role is intentionally separated from `patch_reboot` because:

- Guest agent availability is a **precondition**, not a side effect
- Reboot logic must assume the agent is already functional
- Failures should surface *before* any disruptive action

If this role fails, patching does not proceed.

---

## How This Role Is Used

### Typical Invocation

```yaml
roles:
  - qemu_ga_fix
  - patch_reboot
```

Execution order matters:
1. Guest agent verified and started
2. OS patched
3. Reboot triggered if required

---

## Validation Commands

Before patching:

```bash
ansible TARGET -a "systemctl status qemu-guest-agent"
```

Expected result:
- Service state: `active (running)`
- Enabled on boot

After reboot:

```bash
ansible TARGET -a "systemctl is-active qemu-guest-agent"
```

Expected output:

```text
active
```

---

## Failure Modes & Handling

### Common Failures

| Failure | Cause |
|------|------|
| Package install fails | Apt lock or repo issue |
| Service inactive | VM rebooted without GA |
| Proxmox shows VM hung | GA missing or stopped |

---

### Recovery Steps

On the affected VM:

```bash
sudo apt update
sudo apt install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

If the VM does not respond, use the Proxmox console for recovery.

---

## Safety Guarantees

This role guarantees:

- GA package presence
- GA service is enabled
- GA service is running before patching

It does **not**:
- Trigger reboots
- Modify Proxmox configuration
- Affect bare-metal systems

---

## Scope Limitations

This role applies **only** to:
- Virtual machines running under Proxmox
- Ubuntu-based guests

It must **not** be applied to:
- Bare-metal hosts
- Containers
- Non-systemd environments

---

## Summary

The `qemu_ga_fix` role removes ambiguity from VM lifecycle management. By enforcing guest agent availability before any reboot-capable action, it stabilizes patching workflows and prevents misleading failure states.

Any VM that cannot pass this role is not considered safe to patch automatically.

---
