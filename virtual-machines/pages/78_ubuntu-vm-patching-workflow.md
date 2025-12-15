---

This page documents the **standard patching workflow for Ubuntu-based virtual machines**. This workflow represents the most common maintenance operation in the environment and serves as the reference implementation for safe, repeatable VM patching.

Ubuntu VMs are the only systems intentionally patched **in parallel**, and only because their blast radius is constrained by design.

---

## Scope

This workflow applies to:

- Ubuntu virtual machines
- Non-clustered services
- Stateless or recoverable services
- VMs managed by Proxmox with QEMU Guest Agent support

Examples include:
- Grafana
- BookStack
- Pi-hole (primary and secondary)
- Certificate Authority
- Reverse proxy services

This workflow **does not apply** to:
- Hypervisors
- Backup infrastructure
- Storage systems
- The Ansible control node

---

## Playbook Location

```text
playbooks/patch_all.yml
```

This playbook is intentionally minimal and delegates all logic to roles.

---

## Playbook Content

```yaml
# playbooks/patch_all.yml

- name: Patch and reboot Ubuntu VMs
  hosts: ubuntu_vms
  gather_facts: yes
  become: yes

  roles:
    - qemu_ga_fix
    - patch_reboot

  post_tasks:
    - name: Run Pi-hole health checks
      include_role:
        name: pihole_health
      when: "'pihole' in inventory_hostname"

    - name: Send patch completion notification
      include_role:
        name: notify_email
```

---

## Execution Order

The execution order is deliberate and enforced by role order:

1. **Gather facts**
2. **Ensure QEMU Guest Agent is installed and running**
3. **Apply OS patches**
4. **Reboot only if required**
5. **Run service-specific health checks**
6. **Send notification**

Each host progresses independently through this sequence.

---

## Role Responsibilities

### `qemu_ga_fix`

- Ensures the guest agent is present and active
- Prevents false reboot failures
- Must succeed before patching continues

If this role fails, the playbook stops for that host.

---

### `patch_reboot`

- Applies OS updates
- Performs cleanup
- Reboots only when required
- Waits for SSH connectivity to return

This role is the **only source of reboots** in the workflow.

---

### `pihole_health` (Conditional)

Executed only when the host name contains `pihole`.

Responsibilities:
- Validate DNS resolution
- Confirm service availability
- Detect silent failures caused by reboot

This role is intentionally excluded from non-DNS hosts.

---

### `notify_email`

- Sends completion status
- Includes host-level success or failure
- Provides visibility without console access

Notifications are sent regardless of whether a reboot occurred.

---

## Pre-Flight Validation

Before executing this playbook, the following checks are performed manually to establish baseline state:

```bash
ansible ubuntu_vms -m ping
ansible ubuntu_vms -a "df -h /"
ansible ubuntu_vms -a "uptime -p"
ansible ubuntu_vms -a "who -b"
```

For virtual machines:

```bash
ansible ubuntu_vms -a "systemctl status qemu-guest-agent"
```

If any host fails these checks, it is excluded until resolved.

---

## Execution Command

```bash
ansible-playbook playbooks/patch_all.yml
```

To target a single VM:

```bash
ansible-playbook playbooks/patch_all.yml --limit hostname
```

To perform a dry run:

```bash
ansible-playbook playbooks/patch_all.yml --check
```

---

## Post-Patch Verification

After execution, each host is verified explicitly:

```bash
ansible ubuntu_vms -a "uptime -p"
ansible ubuntu_vms -a "who -b"
ansible ubuntu_vms -a "systemctl is-active qemu-guest-agent"
```

Expected outcomes:
- Uptime reflects recent reboot where applicable
- Boot timestamp aligns with patch window
- Guest agent is active

---

## Failure Modes & Handling

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| Host unreachable post-reboot | Guest agent failure |
| Playbook stalls | SSH reconnect delay |
| Service down | App-level restart required |
| Apt lock | Concurrent package process |

---

### Recovery Approach

- Failed hosts are isolated using `--limit`
- Console access is used if SSH does not return
- The playbook is safe to re-run after recovery

No automatic retries are performed.

---

## Safety Characteristics

This workflow guarantees:

- No infrastructure hosts are touched
- Each VM reboots at most once
- Hosts do not depend on each other
- Failures are visible and localized

Parallelism exists only within a safe boundary.

---

## What This Workflow Does Not Do

- Restart clustered services in coordination
- Patch non-Ubuntu systems
- Perform application upgrades beyond OS packages
- Modify service configuration

Those concerns are handled separately.

---

## Summary

The Ubuntu VM patching workflow represents the **baseline operational model** for the environment. Its structure favors clarity and safety over speed, and it serves as the template for all other patching workflows.

If a system cannot safely fit into this model, it is intentionally excluded.

---
