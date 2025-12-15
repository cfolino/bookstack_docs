---

This page documents the **patching and reboot workflow for the Ansible control node itself**. The control node is treated as **automation infrastructure**, and its maintenance is handled differently from all other systems.

Because this node orchestrates every other workflow, it is **always patched last** and never while automation jobs are actively running.

---

## Scope

This workflow applies **only** to:

- The Ansible control node
- The system hosting playbooks, inventory, and SSH keys
- The system executing Semaphore jobs (if applicable)

This workflow **does not apply** to:
- Application VMs
- Proxmox VE
- Proxmox Backup Server
- OpenMediaVault

---

## Design Constraints

Patching the control node follows strict rules:

- Never patched concurrently with any other system
- Never patched while Semaphore jobs are active
- Always executed manually
- Reboot must be explicitly observed

Loss of the control node mid-run would interrupt all automation, so this workflow prioritizes **control and observability** over convenience.

---

## Playbook Location

```text
playbooks/patch_ansible.yml
```

This playbook is intentionally minimal and relies on the shared patching role.

---

## Playbook Content

```yaml
# playbooks/patch_ansible.yml

- name: Patch and reboot Ansible control node
  hosts: ansible
  gather_facts: yes
  become: yes

  roles:
    - patch_reboot
```

No preparatory roles are included because:
- The control node is not a VM dependency
- No guest agent is required
- Patch logic must remain minimal

---

## Pre-Flight Validation

Before executing the playbook, the following checks are performed manually on the control node:

```bash
uptime -p
who -b
df -h /
```

Additional operational checks:

- Confirm no Semaphore jobs are running
- Confirm no playbooks are executing
- Confirm SSH access from a secondary system if available

These checks ensure the node can be safely rebooted.

---

## Execution Command

From the control node itself:

```bash
ansible-playbook playbooks/patch_ansible.yml
```

Dry run (syntax and role resolution only):

```bash
ansible-playbook playbooks/patch_ansible.yml --check
```

The dry run confirms that:
- Inventory resolution is correct
- Role dependencies are intact
- No unexpected changes are introduced

---

## Reboot Behavior

Reboot behavior is governed by the shared `patch_reboot` role:

- Reboot occurs only if required by the OS
- Only a single reboot is performed
- Ansible waits for SSH to return

Because the playbook runs locally, SSH reconnect behavior is carefully observed.

---

## Post-Patch Verification

After the system returns:

```bash
uptime -p
who -b
ansible --version
```

Expected results:
- Uptime reflects recent reboot (if required)
- Boot time aligns with maintenance window
- Ansible binaries and Python environment are intact

---

## Failure Modes & Recovery

### Common Failure Scenarios

| Symptom | Likely Cause |
|------|------|
| SSH unavailable | Reboot still in progress |
| Ansible fails to start | Python or dependency issue |
| Semaphore unreachable | Service startup delay |

---

### Recovery Strategy

- Access system console if SSH does not return
- Validate disk and filesystem state
- Restart automation services manually if needed
- Re-run the playbook only after validation

Because this is the control plane, recovery is always **manual and deliberate**.

---

## Safety Guarantees

This workflow guarantees:

- The control node is patched in isolation
- No automation jobs are interrupted mid-run
- Reboot logic remains centralized
- Failure is immediately visible

---

## What This Workflow Does Not Do

- Patch any other system
- Manage Semaphore jobs
- Restart unrelated services
- Perform environment upgrades

Those concerns are handled independently.

---

## Summary

The Ansible control node is patched conservatively and intentionally. This workflow reflects the principle that **automation infrastructure must be more stable than the systems it manages**.

If the control node cannot safely complete these steps, patching is postponed until conditions are acceptable.

---
