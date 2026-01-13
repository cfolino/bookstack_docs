## Purpose

This page documents how a **newly provisioned host** is added into the **Day-2 patching and reboot cycle**.

At this stage, the host:
- Is already reachable via SSH
- Has the `ansible` automation user configured
- Has completed Day-1 bootstrap successfully

Day-2 onboarding ensures the system can be **safely patched, rebooted, verified, and scheduled** without manual intervention.

---

## Day-2 Requirements (Gate Checks)

Before a host is eligible for patching automation, the following **must be true**:

- QEMU Guest Agent installed and running
- SSH access via automation user
- Sudo access confirmed
- Reboot behavior is observable and recoverable
- Host is explicitly included in inventory

Automation **does not assume** these conditions — they are verified.

---

## Step 1 — Inventory Registration

Add the host to the Ansible inventory.

Example:
```ini
newhost ansible_host=192.168.x.x
```

Verification:
```bash
ansible newhost -m ping
```

Expected result:
- `pong`
- No authentication fallback or password prompt

---

## Step 2 — QEMU Guest Agent Validation

QEMU Guest Agent is required for safe lifecycle control.

Verification (from Proxmox host):
```bash
qm agent <VMID> ping
```

Expected result:
- Command returns successfully

Guest verification (inside VM):
```bash
systemctl status qemu-guest-agent
```

Expected state:
- `active (running)`

If QGA is not active, the host is **not eligible** for unattended patching.

---

## Step 3 — Patch Role Eligibility Check

Confirm the host matches the expected OS family and patch role behavior.

Verification:
```bash
ansible newhost -m setup | grep ansible_distribution
```

Ensure the host is compatible with:
- `patch_reboot`
- `qemu_ga_fix`
- Any role-specific health checks (DNS, infra, app)

---

## Step 4 — Dry Validation (Read-Only)

Run a targeted validation without triggering patching.

```bash
ansible-playbook playbooks/patch_all.yml \
  --limit newhost \
  --tags verify
```

Expected behavior:
- All verification tasks pass
- No reboot scheduled
- No rescue block triggered

---

## Step 5 — Live Patching Validation

Perform a one-time live patch and reboot to confirm end-to-end behavior.

```bash
ansible-playbook playbooks/patch_all.yml --limit newhost
```

Expected behavior:
- Packages updated
- Reboot performed if required
- Host returns online
- No manual intervention required

---

## Step 6 — Post-Reboot Confirmation

Confirm the host completed a clean reboot.

```bash
ansible newhost -a "uptime"
```

Expected:
- Uptime reflects recent reboot
- SSH responsiveness restored
- No hanging or partial states

---

## Step 7 — Semaphore Job Creation (Per-Host)

Each host receives its **own Semaphore job**, even though all jobs use the same playbook.

### Job Template

- Playbook: `playbooks/patch_all.yml`
- Inventory: Production
- Limit: `newhost`
- Environment: Production
- Notifications: Failure only

The job limit ensures:
- One host is patched at a time
- Reboots are isolated
- Failures do not cascade

---

### Schedule Configuration

Scheduling is defined **per host**.

Typical configuration:
- Frequency: Monthly
- Window: Maintenance-defined (off-hours)
- Offset times are used to prevent overlap

Example cron expression:
```text
0 2 6 1,4,7,10 *
```

Each host’s job may be offset by 15–30 minutes depending on criticality.

---

### First Scheduled Run Expectation

The first scheduled run after onboarding should:
- Execute without manual intervention
- Match the behavior of the manual validation run
- Produce no warnings or rescue actions

If the scheduled job fails:
- The job is disabled
- Root cause is addressed
- A manual validation run is completed
- The job is re-enabled

---

## Failure Handling

If patching fails at any stage:
- Automation enters rescue logic
- The Semaphore job reports failure
- Notification is generated
- Manual review is required before re-enabling scheduling

No silent failures are permitted.

---

## Completion Criteria

A host is considered **fully onboarded for Day-2 patching** when:

- Inventory entry exists and is reachable
- QGA is verified
- Patch and reboot cycle completes once successfully
- Host-specific Semaphore job executes successfully
- Post-reboot health checks pass

At this point, the host becomes a **first-class participant** in scheduled maintenance.

---

## Notes

- One Semaphore job per host is intentional
- Isolation is preferred over parallelism
- Scheduling is explicit and auditable
- All jobs share the same playbook logic

This keeps patching predictable, recoverable, and easy to reason about.
