---

This page documents the **central patching and reboot role** used across the environment.
The `patch_reboot` role is the **only component permitted to reboot a system**. This constraint is intentional and non-negotiable.

All other roles that perform validation, preparation, or post-checks **must not** trigger reboots.

---

## Role Purpose

The `patch_reboot` role provides a **consistent, minimal, and auditable** mechanism for:

- Updating OS packages
- Cleaning up obsolete packages
- Determining whether a reboot is required
- Performing a single, controlled reboot
- Waiting for the system to return online

This role abstracts OS-level maintenance so higher-level playbooks remain simple and predictable.

---

## Role Location

```text
roles/patch_reboot/
├── tasks/
│   └── main.yml
└── defaults/
    └── main.yml
```

---

## Role Defaults

```yaml
# roles/patch_reboot/defaults/main.yml
reboot_timeout: 900
```

### Purpose of Defaults
- Provides a safe reboot window for kernel upgrades
- Can be overridden at group or host level if needed
- Prevents hardcoded timeouts inside tasks

---

## Task Breakdown

### Task File

```yaml
# roles/patch_reboot/tasks/main.yml

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
  become: true

- name: Apply available upgrades
  ansible.builtin.apt:
    upgrade: dist
    autoremove: yes
    autoclean: yes
  become: true

- name: Reboot if required
  ansible.builtin.reboot:
    reboot_timeout: "{{ reboot_timeout }}"
  when: ansible_facts.reboot_required | default(false)
```

---

## Task Explanation

### Update Apt Cache

```yaml
ansible.builtin.apt:
  update_cache: yes
```

Ensures the package index is current before attempting upgrades.
This prevents partial or stale updates.

---

### Apply OS Upgrades

```yaml
ansible.builtin.apt:
  upgrade: dist
  autoremove: yes
  autoclean: yes
```

Behavior:
- Applies kernel and dependency updates
- Removes unused packages
- Cleans package cache

Rationale:
- `dist-upgrade` is required for kernel and dependency transitions
- Cleanup reduces disk pressure on small root filesystems

---

### Conditional Reboot

```yaml
when: ansible_facts.reboot_required | default(false)
```

Reboot logic is **strictly conditional**.

- No reboot occurs unless the OS explicitly requires it
- Prevents unnecessary service disruption
- Ensures idempotent behavior

The presence of `/var/run/reboot-required` is evaluated by Ansible facts.

---

## Execution Behavior

When executed:

- Hosts that do not require a reboot remain online
- Hosts that require a reboot reboot exactly once
- Ansible waits for SSH connectivity to return
- Failure to reconnect is treated as a hard failure

---

## How This Role Is Invoked

### Example: Ubuntu VM Patching

```yaml
roles:
  - qemu_ga_fix
  - patch_reboot
```

The reboot always occurs **after** preparatory roles complete.

---

## Pre-Conditions

Before this role is run:

- SSH access must be functional
- Disk space must be sufficient for upgrades
- Any required preparatory roles must have completed successfully

These checks are enforced by upstream playbooks.

---

## Post-Conditions

After this role completes:

- System packages are fully upgraded
- Reboot state is resolved
- Host is reachable via SSH
- Control returns cleanly to Ansible

No assumptions are made about application state.

---

## Failure Modes & Handling

### Common Failures

| Failure | Cause |
|------|------|
| Apt lock | Concurrent package process |
| Reboot timeout | Slow hardware or kernel update |
| SSH unreachable | Guest agent or boot issue |

---

### Recovery Guidance

```bash
# On the affected host
sudo dpkg --configure -a
sudo apt -f install
```

If a host does not return after reboot, console access is required.

---

## Safety Guarantees

This role guarantees:

- One reboot per execution
- No nested reboots
- No silent failures
- No cross-host dependencies

It does **not** guarantee application availability — that is handled elsewhere.

---

## What This Role Does Not Do

- It does not manage services
- It does not validate application health
- It does not patch non-OS software
- It does not modify configuration files

Those responsibilities are intentionally outside its scope.

---

## Summary

The `patch_reboot` role is the **single source of truth** for OS patching and reboots.
Its simplicity is deliberate — complexity is handled by orchestration, not the role itself.

Any automation that needs a reboot must depend on this role or not reboot at all.

---
