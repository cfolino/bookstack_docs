---

This page documents the **inventory design**, **group hierarchy**, and **variable scoping model** used by the Ansible automation framework.

The inventory is not a static host list — it is the **control plane abstraction** that enables safe orchestration, delegation, and role reuse across the homelab.

---

## Inventory Design Goals

The inventory is designed to:

- Reflect **infrastructure reality**, not convenience
- Enable **role reuse** without conditionals
- Support **safe delegation** (Proxmox, CA, localhost)
- Allow **selective orchestration** by function
- Scale cleanly as hosts are added

It intentionally avoids:
- Flat host lists
- Environment duplication
- Host-specific logic in playbooks

---

## Inventory Layout

```
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

This structure enforces **separation of concerns**:

- `hosts.yml` → topology and grouping
- `group_vars/` → shared behavior
- `host_vars/` → identity and orchestration metadata

---

## Inventory File (`hosts.yml`)

The inventory defines **functional groupings**, not environments.

### Core Groups

```
all:
  children:
    infrastructure:
      hosts:
        pve:
        pbs:
        ansible:
        omv
```

**Infrastructure** hosts are foundational services that:
- Host other systems
- Orchestrate or store data
- Require careful reboot sequencing

---

### Ubuntu Virtual Machines

```
ubuntu_vms:
  hosts:
    bookstack:
    grafana:
    npm:
    pihole:
    pihole-backup:
    ca
```

This group exists to:
- Apply common OS-level automation
- Patch and reboot safely
- Deploy exporters consistently

No service logic is embedded here — only shared OS traits.

---

### Functional Service Groups

```
dns:
  hosts:
    pihole:
    pihole-backup

proxy:
  hosts:
    npm

certifcate_authority:
  hosts:
    ca
```

These groups enable:
- Targeted playbooks
- Role isolation
- Clear ownership boundaries

They are **orthogonal** to OS or infrastructure grouping.

---

## Group Membership Philosophy

Hosts may belong to **multiple groups** simultaneously.

Example:

- `pbs` belongs to:
  - `infrastructure`
  - `ubuntu_vms`

This allows:
- Inclusion in global patching
- Inclusion in infra-aware orchestration
- Explicit targeting when needed

No group is mutually exclusive.

---

## Host Variables (`host_vars/`)

Each host has a dedicated variable file that defines **identity**, not behavior.

### Example: `host_vars/pbs.yml`

```
vmid: 101
```

### Why VMID Lives Here

- VMID is **not a service attribute**
- It is an **orchestration identifier**
- Required for:
  - Proxmox delegation
  - Controlled reboots
  - VM lifecycle actions

Keeping it in `host_vars` ensures:
- No hardcoding in roles
- Safe reuse of reboot logic
- Clean delegation to Proxmox

---

## Delegation & Control Flow

The inventory enables controlled delegation patterns:

### Proxmox Delegation

- Playbook targets VM
- Reboot delegated to `pve`
- Uses VMID from `host_vars`

```
delegate_to: pve
command: qm reboot {{ vmid }}
```

This would be impossible without structured inventory metadata.

---

### Certificate Authority Delegation

- Playbook targets `npm`
- CSR and signing delegated to `ca`
- Artifacts fetched via controller

Inventory defines **trust paths**, not just reachability.

---

## Group Variables (`group_vars/all.yml`)

Global variables define **shared infrastructure constants**, such as:

- SMTP behavior
- SSL paths
- Shared defaults

They never include:
- Secrets
- Host identity
- Execution logic

This prevents variable shadowing and ambiguity.

---

## Inventory as a Safety Mechanism

This inventory structure prevents:

- Accidental reboots of wrong hosts
- Certificate deployment to unintended systems
- Role execution without required metadata
- Cross-service contamination

Playbooks fail fast when:
- Required groups are missing
- Required host vars are undefined

This is intentional.

---

## Operational Verification

To visualize the active topology:

```
ansible-inventory -i inventory/hosts.yml --graph
```

Expected behavior:
- Clear group nesting
- No orphaned hosts
- Predictable membership

---

## Why This Inventory Works

This design:

- Mirrors real infrastructure relationships
- Enables complex orchestration safely
- Keeps playbooks readable
- Keeps roles generic
- Keeps control centralized

It scales by **addition**, not refactor.

---

## Summary

The inventory is the **foundation** of the automation framework.

It defines:
- What exists
- How systems relate
- Where control boundaries live

Everything else — patching, certificates, monitoring, orchestration — builds on this layer.

---

## Related Pages

- Ansible Control Node Bootstrap
- Patch & Reboot Orchestration
- Proxmox VE Automation
- Certificate Issuance & Deployment
- Automation Email Architecture

---

**If the inventory is correct, automation becomes predictable.**
