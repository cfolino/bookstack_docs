---

This page documents how **Semaphore** is integrated into the patching framework to provide scheduling, access control, and execution visibility without altering playbook behavior or introducing hidden automation.

Semaphore is treated as an **execution layer**, not a logic layer.

---

## Purpose

Semaphore is used to:

- Schedule routine patching windows
- Provide role-based access control (RBAC)
- Capture execution history and logs
- Allow safe delegation of automation execution

Semaphore does **not** change how patching works — it only controls **when and by whom** playbooks are run.

---

## Design Philosophy

The integration follows strict principles:

- Playbooks must run identically via CLI or Semaphore
- No Semaphore-specific conditionals exist in roles
- No logic depends on Semaphore variables
- Semaphore failures must not mask Ansible failures

This guarantees that Semaphore can be removed without refactoring automation.

---

## What Semaphore Controls

Semaphore is responsible for:

- Job scheduling
- Inventory selection
- Environment variable injection (where needed)
- Access control to playbooks
- Retention of execution logs

Semaphore is **not** responsible for:
- Patch logic
- Reboot logic
- Health checks
- Failure recovery

---

## Semaphore Job Types

Typical jobs mapped to patching workflows include:

| Job Name | Playbook |
|--------|---------|
| Patch Ubuntu VMs | `patch_all.yml` |
| Patch Proxmox VE | `patch_pve.yml` |
| Patch PBS | `patch_pbs.yml` |
| Patch OMV | `patch_omv.yml` |
| Patch Ansible Node | `patch_ansible.yml` |

Each job maps to **one playbook** and one operational scope.

---

## Job Configuration Principles

Each Semaphore job is configured with:

- A single playbook
- The production inventory
- No ad-hoc extra variables unless required
- Explicit execution limits where appropriate

Example execution limit:

```text
--limit grafana
```

Limits are used for targeted maintenance, not bulk safety.

---

## Environment Handling

When required, Semaphore injects environment variables for:

- SMTP configuration
- Certificate trust chains
- API access

These variables are treated as **inputs**, not logic flags.

Example:

```text
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

No playbook behavior changes based on whether Semaphore is present.

---

## Execution Behavior

When a job is triggered:

- Semaphore checks out the repository
- Ansible executes the playbook
- Full stdout/stderr is captured
- Job status reflects Ansible result

Success or failure is determined exclusively by Ansible.

---

## Logging & History

Semaphore provides:

- Per-job execution history
- Per-run logs
- Execution timestamps
- Operator attribution

This augments, but does not replace, Ansible output.

---

## Failure Handling

If a job fails:

- Semaphore marks the run as failed
- Logs remain available for inspection
- No automatic retries are performed
- No remediation is triggered

Failures are investigated and resolved manually.

---

## Control Node Safeguards

Special handling applies to the Ansible control node:

- Control node patching is not scheduled
- Control node jobs are executed manually
- Semaphore is not used to patch itself

This prevents self-hosted automation failure loops.

---

## Security Considerations

- Access to jobs is role-restricted
- Secrets are stored in Semaphore, not playbooks
- Inventory modification is restricted
- Execution permissions are explicit

Semaphore is treated as a privileged interface.

---

## Validation & Parity Testing

Playbooks are periodically tested via CLI to confirm parity:

```bash
ansible-playbook playbooks/patch_all.yml
```

If a playbook behaves differently under Semaphore, it is considered a defect.

---

## What Semaphore Is Not Used For

- Writing or editing playbooks
- Making patch decisions
- Handling reboots
- Performing emergency recovery
- Modifying inventory structure

Those responsibilities remain with Ansible and operational processes.

---

## Summary

Semaphore provides **controlled execution and visibility** without becoming a dependency for logic. The patching framework remains portable, transparent, and deterministic regardless of how it is executed.

If Semaphore is unavailable, all automation remains functional.

---
