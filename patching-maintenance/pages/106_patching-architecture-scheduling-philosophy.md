This page documents the monthly patching system built around a clear separation of concerns:

- **GitHub** is the source of truth for code.
- **Semaphore** schedules and executes patch jobs.
- **Ansible** performs patching and reboot logic.

The design emphasizes **safety, predictability, and auditability** over aggressive automation.

---

## Architecture Overview

### GitHub (authoritative code)
GitHub holds the canonical versions of:

- Playbooks
- Roles
- Inventory structure
- Host variables

Scheduling logic is not embedded in Ansible. Timing and spacing decisions are owned by Semaphore.

### Semaphore (scheduler + executor)
Semaphore is responsible for:

- Defining monthly schedules
- Staggering execution to avoid overlap
- Running one job per VM (intentional isolation)
- Providing execution history for audit trails

Semaphore executes Ansible directly. There is no intermediate scheduler layer.

### Ansible (implementation engine)
Ansible is responsible for:

- Applying OS updates
- Handling reboots (only when required by the playbook/role logic)
- Performing pre/post validation checks
- Producing consistent outcomes regardless of whether runs are manual or scheduled

Playbooks and roles were **not redesigned for scheduling**. The scheduling model wraps existing automation safely without changing its intent.

---

## Scheduling Philosophy

The scheduling model is conservative by design:

- **Monthly cadence** is used for routine patching.
- Jobs are **staggered** to avoid overlap and reduce resource contention.
- Each VM is isolated into a **single dedicated job** to make failures attributable and easy to troubleshoot.
- **DNS is treated as critical infrastructure** and is handled with stronger separation rules than other VMs.

This approach prefers controlled, repeatable maintenance windows over parallel execution.

---

## Explicit Scope Boundaries

This monthly patching system currently covers:

- Ubuntu VMs patched via Ansible and scheduled via Semaphore
- DNS infrastructure with special scheduling separation

The following hosts are explicitly excluded and documented elsewhere:

- Proxmox VE (PVE)
- Proxmox Backup Server (PBS)
- OpenMediaVault (OMV)

Those systems are treated as stateful infrastructure with separate safety requirements.

---

## Intent

This system is built to be:

- Easy to reason about
- Safe to run unattended during maintenance windows
- Auditable after the fact through Semaphore job history
- Extensible later without forcing a redesign of working playbooks
