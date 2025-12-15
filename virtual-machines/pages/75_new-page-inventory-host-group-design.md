---

This page documents how the Ansible inventory is structured to **enforce safety, scope control, and predictable patching behavior** within the `cfolino.com` homelab.

The inventory is not treated as a passive host list. It is the **primary safety boundary** that determines what systems can be patched together and which automation paths apply.

---

## Design Intent

The inventory is structured to:

- Encode operational and trust boundaries
- Prevent accidental cross-tier patching
- Make patch scope explicit and reviewable
- Avoid conditional logic inside playbooks
- Support incremental growth without refactoring

Incorrect inventory placement is considered a configuration error, not a runtime issue.

---

## Inventory Location & Layout

The inventory is maintained at:

```text
/home/ansible/ansible/inventory/hosts.yml
```

Supporting files:

```text
/home/ansible/ansible/inventory/
├── hosts.yml
├── group_vars/
└── host_vars/
```

This layout allows:
- Global defaults (`group_vars`)
- Host-specific overrides (`host_vars`)
- Clear separation of structure and behavior

---

## Top-Level Inventory Structure

```yaml
all:
  children:

    infrastructure:
      hosts:
        pve:
        pbs:
        ansible:
        omv:

    ubuntu_vms:
      hosts:
        bookstack:
        grafana:
        npm:
        pihole:
        pihole-backup:
        ca:
```

This structure deliberately separates **infrastructure systems** from **application VMs**.

---

## Inventory Groups and Their Meaning

### `infrastructure`

This group contains systems where downtime has **broad or cascading impact**.

Members include:
- Proxmox VE (hypervisor)
- Proxmox Backup Server
- OpenMediaVault (storage)
- Ansible control node

Rules:
- Never patched in bulk
- Never combined with other groups
- Always patched using dedicated playbooks

---

### `ubuntu_vms`

This group contains general-purpose Ubuntu virtual machines.

Members include:
- Monitoring
- Documentation
- DNS services
- Certificate authority
- Reverse proxy services

Rules:
- Can be patched together
- Reboots are independent per host
- Guest agent must be active

---

## Host Naming & Resolution

Each host entry resolves to an internal FQDN:

```yaml
grafana:
  ansible_host: <internal-host>
```

Design constraints:
- Inventory names are short and stable
- DNS provides authoritative resolution
- IP addresses are not hardcoded

This prevents IP drift from breaking automation.

---

## Adding a New Host to Inventory

### Step 1 – Determine the Correct Group

Before adding a host, its operational role must be identified:

| System Type | Inventory Group |
|-----------|-----------------|
| Ubuntu VM | `ubuntu_vms` |
| Hypervisor | `infrastructure` |
| Backup | `infrastructure` |
| Storage | `infrastructure` |
| Control Plane | `infrastructure` |

If classification is unclear, the host is not added.

---

### Step 2 – Add the Host Entry

```yaml
ubuntu_vms:
  hosts:
    newhost:
      ansible_host: <internal-host>
```

The group assignment controls which patching logic applies.

---

### Step 3 – Validate Inventory Integrity

```bash
ansible newhost -m ping
```

Expected result confirms:
- Inventory syntax
- SSH access
- DNS resolution

---

## Group Variables (`group_vars`)

Group-level behavior is defined in:

```text
inventory/group_vars/
```

Examples:
- Default reboot timeouts
- Notification recipients
- Environment-specific flags

Group variables ensure consistent behavior across hosts of the same class.

---

## Host Variables (`host_vars`)

Host-specific overrides are stored in:

```text
inventory/host_vars/
```

Use cases:
- Non-standard reboot timeouts
- Unique service checks
- Temporary maintenance constraints

Host variables are used sparingly and intentionally.

---

## Inventory as a Safety Mechanism

The inventory is intentionally opinionated:

- Playbooks do not override group boundaries
- Roles assume inventory correctness
- Safeguards rely on proper grouping

If a host does not belong in a group, it does not belong in the framework.

---

## Validation & Dry Runs

Before executing any patching playbook:

```bash
ansible-playbook playbooks/patch_all.yml --check
```

Inventory errors are surfaced early and treated as blocking issues.

---

## What the Inventory Does Not Do

- It does not encode execution order
- It does not perform health checks
- It does not trigger reboots
- It does not manage secrets

Those responsibilities belong to roles and playbooks.

---

## Summary

The inventory defines **what is allowed to happen** before any automation runs. It enforces operational boundaries that are not negotiable at runtime.

If the inventory is correct, the automation behaves safely.
If it is not, the automation is not run.

---
