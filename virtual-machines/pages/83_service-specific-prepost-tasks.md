---

This page documents the **service-specific validation tasks** that are executed before or after patching workflows. These tasks exist to catch **silent service failures** that OS-level patching and reboot verification alone cannot detect.

Service checks are intentionally **decoupled** from core patching logic to preserve modularity and reuse.

---

## Purpose

Service-specific tasks are used to:

- Validate critical services after reboot
- Detect failures that do not break SSH connectivity
- Provide targeted assurance without global complexity
- Keep the core patching role generic and reusable

These tasks are **supplemental**, not mandatory for every host.

---

## Design Principles

Service-specific tasks follow these rules:

- Never perform reboots
- Never modify configuration
- Never assume other services are present
- Execute conditionally based on host identity
- Fail fast and visibly when checks do not pass

They are designed to **observe**, not to remediate.

---

## Where These Tasks Are Defined

Service checks are implemented as **separate roles** and included conditionally.

Example locations:

```text
roles/
├── pihole_health/
│   └── tasks/main.yml
├── grafana_health/
│   └── tasks/main.yml
└── bookstack_health/
    └── tasks/main.yml
```

Each role is responsible only for its own service.

---

## How Service Tasks Are Invoked

Service-specific roles are included using conditional logic in the playbook.

### Example (Ubuntu VM Patching)

```yaml
post_tasks:
  - name: Run Pi-hole health checks
    include_role:
      name: pihole_health
    when: "'pihole' in inventory_hostname"
```

This ensures:
- The role runs only where applicable
- No assumptions are made about unrelated hosts

---

## Example: Pi-hole Health Checks

### Role Location

```text
roles/pihole_health/tasks/main.yml
```

### Role Tasks

```yaml
- name: Verify Pi-hole DNS resolution
  ansible.builtin.command: dig google.com @127.0.0.1
  register: dns_test
  failed_when: dns_test.rc != 0

- name: Verify Pi-hole service status
  ansible.builtin.systemd:
    name: pihole-FTL
    state: started
```

---

## Task Explanation

### DNS Resolution Test

```yaml
dig google.com @127.0.0.1
```

Confirms:
- DNS service is responding
- Local resolver is functional
- Networking survived the reboot

A successful exit code is required.

---

### Service State Check

```yaml
ansible.builtin.systemd:
  name: pihole-FTL
  state: started
```

Ensures the service is active after reboot.

This task does **not** restart services arbitrarily; it asserts expected state.

---

## Example: Grafana Health Check

### Role Tasks

```yaml
- name: Verify Grafana service
  ansible.builtin.systemd:
    name: grafana-server
    state: started

- name: Verify Grafana HTTP endpoint
  ansible.builtin.uri:
    url: https://internal.example
    status_code: 200
```

These checks validate both:
- Service-level availability
- Application-level responsiveness

---

## Example: BookStack Health Check

### Role Tasks

```yaml
- name: Verify BookStack web service
  ansible.builtin.uri:
    url: https://internal.example
    status_code: 200
```

This confirms:
- Web server is reachable
- Application stack is responding

---

## Pre-Task vs Post-Task Usage

### Pre-Tasks

Used when:
- A service must be quiesced
- A known state must be confirmed before reboot

Example use cases:
- Backup job inactivity
- Cluster quorum state

---

### Post-Tasks

Used when:
- Service availability must be confirmed after reboot
- Silent failures are possible

Most service checks are implemented as post-tasks.

---

## Execution Characteristics

Service-specific tasks:

- Run only on applicable hosts
- Do not affect other hosts
- Fail the playbook for that host only
- Are safe to re-run independently

Failures do not automatically trigger remediation.

---

## Failure Handling Philosophy

When a service-specific check fails:

- The failure is surfaced immediately
- The host is considered patched but degraded
- Manual investigation is required
- Automation does not attempt blind recovery

This prevents automation from masking real issues.

---

## What These Tasks Do Not Do

- Restart services repeatedly
- Modify application configuration
- Apply patches or upgrades
- Influence reboot behavior
- Coordinate cross-host services

Those concerns are intentionally excluded.

---

## Summary

Service-specific pre/post tasks provide **application-level assurance** without compromising the simplicity of the patching framework. They act as targeted sensors layered on top of OS maintenance.

If a service requires more than validation, it does not belong in this layer.

---
