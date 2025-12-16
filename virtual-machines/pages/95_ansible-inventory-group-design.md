---

This page documents the structure, intent, and design decisions behind the Ansible inventory used to manage the homelab. It serves as the authoritative reference for how hosts are grouped, how variables are assigned, and how automation targets are selected.

Every Ansible playbook and role in this environment relies on this inventory design.

---

## Inventory Layout

```text
inventory/
├── hosts.yml
├── group_vars/
│   └── all.yml
└── host_vars/
    ├── ansible.yml
    ├── bookstack.yml
    ├── ca.yml
    ├── grafana.yml
    ├── npm.yml
    ├── pbs.yml
    ├── pihole.yml
    └── pihole-backup.yml
```

---

## Inventory File (`hosts.yml`)

The inventory uses **logical grouping**, not OS-only or function-only grouping. Hosts often belong to **multiple groups** to allow precise automation targeting.

### Core Groups

| Group | Purpose |
|-----|--------|
| `infrastructure` | Core systems that underpin the environment |
| `ubuntu_vms` | Ubuntu-based virtual machines |
| `proxy` | Reverse proxy services |
| `dns` | DNS infrastructure |
| `certifcate_authority` | Internal PKI host |
| `unifi` | UniFi networking devices |

---

## Group Membership Strategy

### `infrastructure`
Includes:
- pve
- pbs
- ansible
- omv

Used for:
- Core lifecycle operations
- Monitoring exporters
- Cross-host orchestration

---

### `ubuntu_vms`
Includes:
- bookstack
- grafana
- npm
- pihole
- pihole-backup
- ca

Used for:
- Patch orchestration (`patch_all.yml`)
- Monitoring exporters
- OS-level maintenance

This group deliberately **excludes**:
- Proxmox VE
- OMV
- PBS
(these are patched via specialized roles)

---

### `proxy`
Includes:
- npm

Used for:
- Certificate deployment
- Reverse proxy maintenance
- TLS automation

---

### `dns`
Includes:
- pihole
- pihole-backup

Used for:
- DNS health checks
- Redundancy validation
- Monitoring probes

---

### `certifcate_authority`
Includes:
- ca

Used for:
- Certificate signing
- PKI automation
- Restricted trust operations

---

### `unifi`
Includes:
- udmpro

Used for:
- UniFi API automation
- Network device health checks
- Controlled reboots

---

## Variable Placement Strategy

Variables are placed deliberately to avoid drift and duplication.

---

### `group_vars/all.yml`

Used for:
- Global defaults
- Shared service configuration
- Notification settings

Examples:
- SMTP configuration
- Common paths
- Shared flags

This file should **never** contain host-specific values.

---

### `host_vars/*.yml`

Each managed host has a corresponding `host_vars` file.

Used for:
- VM identifiers (`vmid`)
- Host-specific paths
- Role-specific overrides
- Certificate mappings

Example:
```yaml
vmid: 104
```

This allows:
- Proxmox-level orchestration
- Safe delegated reboots
- Consistent lifecycle automation

---

## VMID Standardization

Every virtual machine managed by Ansible defines a `vmid` in `host_vars`.

This enables:
- Proxmox-level reboot control
- Safe shutdown/start sequences
- Unified behavior across patching roles

Example usage:
```yaml
command: qm reboot {{ vmid }}
delegate_to: pve
```

Bare-metal hosts (OMV, PVE) do **not** rely on VMID-based reboots.

---

## Why Hosts Belong to Multiple Groups

Hosts are grouped by **function**, not exclusivity.

Example:
- `grafana` belongs to:
  - `ubuntu_vms`
  - `internal`

This allows:
- One playbook to install exporters on all Ubuntu VMs
- Another to target only internal services
- Another to isolate monitoring components

This avoids:
- Duplicated playbooks
- Hardcoded host lists
- Inventory sprawl

---

## Inventory Validation

### Visualize Group Membership
```bash
ansible-inventory -i inventory/hosts.yml --graph
```

### Target a Specific Group
```bash
ansible-playbook playbooks/patch_all.yml -l ubuntu_vms
```

---

## Design Principles

- Groups reflect **operational intent**
- Host vars contain **identity**, not logic
- Roles are reusable across groups
- Inventory remains human-readable
- No dynamic inventory complexity (by design)

---

## Why This Matters

This inventory design enables:
- Safe automation at scale
- Clear separation of responsibilities
- Predictable behavior across patching, monitoring, PKI, and networking
- Long-term maintainability without refactoring

Every automation documented elsewhere in BookStack assumes this structure.

---

## Related Pages
- Ansible Control Node Patching
- Patch Orchestration Strategy
- Monitoring Exporters
- Internal Certificate Authority Automation

---
