This page documents how individual VMs are scheduled for monthly patching and records the current active schedule.

The intent is clarity and predictability rather than dynamic orchestration.

---

## Per-VM Scheduling Strategy

Each VM is patched using its **own Semaphore job**.

This approach ensures:

- Clear attribution of success or failure to a single host
- Independent enable/disable control per VM
- Safe iteration when adding or adjusting hosts
- Clean audit history without multi-host ambiguity

Jobs are not grouped by role or function. Even when playbooks are shared, scheduling remains host-specific.

---

## Staggering Model

For non-DNS Ubuntu VMs:

- Jobs are scheduled on the same calendar day
- Start times are staggered by **15 minutes**
- This avoids overlapping reboots and reduces load contention

No automatic dependency or chaining exists between jobs.

---

## DNS Scheduling Isolation

DNS infrastructure follows stricter rules and is documented separately.

Key constraints reflected in the schedule:

- DNS jobs do not overlap with each other
- DNS jobs do not overlap with non-DNS jobs
- PRIMARY and BACKUP DNS servers are patched on different days

These constraints are enforced by scheduling decisions, not automation logic.

---

## Current Monthly Schedule

### Day 2 — Non-DNS Ubuntu VMs

| VM | Start Time (Local) |
|----|--------------------|
| BookStack | 02:00 |
| Grafana | 02:15 |
| Nginx Proxy Manager | 02:30 |
| Certificate Authority (CA) | 02:45 |

All jobs on this day use the same staggered window and do not overlap.

---

### Day 3 — DNS

| VM | Start Time (Local) |
|----|--------------------|
| Pi-hole PRIMARY | 02:00 |

---

### Day 4 — DNS

| VM | Start Time (Local) |
|----|--------------------|
| Pi-hole BACKUP | 02:30 |

---

## Explicit Exclusions

The following hosts are intentionally excluded from this schedule:

- Proxmox VE (PVE)
- Proxmox Backup Server (PBS)
- OpenMediaVault (OMV)

They are treated as stateful infrastructure and are documented in separate maintenance workflows.
