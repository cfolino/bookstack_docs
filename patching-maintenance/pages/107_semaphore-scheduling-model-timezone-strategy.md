This page documents how Semaphore is used to schedule and execute monthly patching jobs, and why specific scheduling decisions were made.

Semaphore owns **when** jobs run.
Ansible owns **what** happens when they run.

---

## Semaphore as the Scheduling Authority

Semaphore is used as the single scheduling authority for patching jobs. It is responsible for:

- Defining the execution calendar
- Enforcing start times and spacing
- Preventing job overlap
- Providing execution history and visibility

No scheduling logic exists in Ansible playbooks or roles. This keeps automation reusable and prevents hidden time-based behavior.

---

## Timezone Configuration

Semaphore is configured globally to use:

- **Timezone:** `America/New_York`

This decision ensures that:

- Maintenance windows align with local expectations
- Job start times are human-readable and predictable
- Logs and notifications match local time without interpretation
- Day-based separation (for DNS) is unambiguous

All scheduling decisions are made in local time. No playbooks perform timezone conversion.

---

## One Job per VM (Intentional Design)

Each VM is patched using a **dedicated Semaphore job**.

This is intentional and provides several advantages:

- Failures are isolated to a single host
- Jobs can be enabled, disabled, or rescheduled independently
- Execution history is clean and host-specific
- No ambiguity exists about which host a job targeted

Even when multiple VMs use the same playbook, jobs are never shared across hosts.

---

## Monthly Cadence

All patching jobs are scheduled on a **monthly cadence**.

This cadence was chosen to:

- Balance security updates with operational stability
- Reduce maintenance noise
- Keep scheduling predictable and reviewable

Ad-hoc or emergency patching can still be triggered manually without altering the monthly schedule.

---

## Why Scheduling Is Not Embedded in Ansible

Scheduling was intentionally excluded from Ansible to avoid:

- Calendar-aware playbook logic
- Conditional execution based on date or time
- Divergence between manual and scheduled runs

By keeping scheduling in Semaphore, Ansible remains deterministic and environment-agnostic.
