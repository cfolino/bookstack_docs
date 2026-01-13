Adding a VM to the monthly patching system is an **intentional, manual process by design**.

This preserves safety, reviewability, and predictable behavior. Automation may be layered later, but is not required for correctness.

---

## Design Intent

The onboarding process is deliberately manual to ensure:

- Scheduling decisions are reviewed by a human
- DNS and critical infrastructure rules are respected
- New hosts do not introduce overlap or risk
- Failures are isolated and attributable

No tooling currently auto-generates jobs or schedules.

---

## Inventory Requirements

Before a VM can be scheduled:

- The host exists in Ansible inventory
- Host grouping is correct (e.g., `ubuntu_vms`)
- SSH access is key-based and functional
- The host is compatible with existing patching roles

Inventory changes do not include scheduling metadata.

---

## Host Variable Expectations

The host must conform to existing expectations, including:

- Correct `ansible_user`
- Appropriate privilege escalation behavior
- Compatibility with patch and reboot logic already in use

No per-host schedule variables are required.

---

## Semaphore Job Creation

To add a VM:

1. Create a **new Semaphore job**
2. Associate the job with:
   - The existing GitHub repository
   - The appropriate Ansible playbook
3. Limit execution to **one host only**
4. Run the job manually once to validate behavior

Jobs are never shared across multiple hosts.

---

## Scheduling & Spacing Considerations

When assigning a schedule:

- Avoid overlap with existing jobs
- Maintain at least **15 minutes** spacing for non-DNS hosts
- Never overlap DNS and non-DNS jobs
- Never schedule PRIMARY and BACKUP DNS on the same day

All spacing decisions are intentional and documented.

---

## DNS Exclusion Rules

If the VM provides or depends on DNS services:

- Treat it as critical infrastructure
- Schedule it independently
- Isolate it from all other patch windows

DNS safety always overrides scheduling convenience.

---

## Verification Checklist

Before enabling the schedule:

- [ ] Manual Semaphore run completes successfully
- [ ] OS updates apply without error
- [ ] Reboot behavior is correct (only if required)
- [ ] Host is reachable post-run
- [ ] Notifications (if configured) are received
- [ ] No overlap with existing scheduled jobs

Only after verification is the schedule enabled.

---

## Future Note

This process is manual by intent.

Additional automation may be introduced later, but only after this model has proven stable and predictable over time.
