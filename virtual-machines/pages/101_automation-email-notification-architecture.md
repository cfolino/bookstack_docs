---

This page documents the **email notification system** used by the Ansible automation framework.
It explains **why notifications exist**, **where they originate**, and **how they are intentionally constrained** to preserve security, clarity, and signal quality.

Email is treated as an **audit and visibility channel**, not an alert storm.

---

## Purpose of Automation Emails

Automation emails exist to:

- Provide **human visibility** into infrastructure changes
- Summarize **what changed**, **where**, and **why**
- Confirm **successful completion** of high-impact operations
- Capture **failure context** without requiring log access

They are not designed for:
- Real-time alerting
- Monitoring replacement
- Ticket automation

Monitoring belongs in **Grafana/Prometheus**.
Email belongs to **change awareness and accountability**.

---

## Centralized Notification Model

### Single Origin Principle

All automation emails originate from **one system only**:

- **Ansible Control Node (`<internal-host>`)**

No other host:
- Sends automation emails
- Contains SMTP credentials
- Knows how to notify externally

This creates:
- A single trust boundary
- A single audit source
- Predictable delivery behavior

---

## Notification Role: `notify_email`

All email delivery is abstracted into a **single reusable role**:

```
roles/notify_email/
```

This role is:
- Stateless
- Idempotent
- Called explicitly by playbooks
- Never auto-triggered

### Why a Role (Not Inline Tasks)

Using a role ensures:
- SMTP logic is centralized
- Formatting is consistent
- Credentials are not duplicated
- Changes affect all notifications uniformly

---

## Email Transport Design

### Transport Method

- SMTP over TLS
- Authenticated relay
- Python-based sender (`sendmail_gmail.py`)

The role uses a **local execution model**:

- Email is always sent from `localhost`
- Remote hosts never connect to SMTP
- Network egress is tightly controlled

---

## Credential Handling

SMTP credentials are:

- Stored centrally
- Referenced via variables
- Never embedded in playbooks
- Never copied to remote hosts

The control node is the **only secret holder**.

If compromised, credentials are rotated in one place.

---

## Email Structure & Content

### Format

- HTML-formatted body
- Structured sections
- Preformatted blocks for logs and output
- Human-readable summaries

### Required Fields

Each notification includes:

- Target host (`inventory_hostname`)
- Operation performed
- Pre- and post-state where relevant
- Reboot or service impact
- Timestamp context

Emails are designed to be **read in under one minute**.

---

## When Emails Are Sent

Emails are intentionally limited to **high-value events**, including:

- Patch & reboot operations
- Proxmox host maintenance
- PBS upgrades
- OMV patch cycles
- Certificate issuance and deployment
- Control node patching

Low-impact or read-only operations do **not** send email.

---

## Playbook Integration Pattern

Email notifications are **explicit**, never implicit.

Typical pattern:

```
- name: Build notification content
  set_fact:
    notify_subject: "..."
    notify_body: "..."

- name: Send notification
  include_role:
    name: notify_email
```

This ensures:
- The playbook controls context
- Emails are readable
- Notification logic never mutates state

---

## Failure Visibility Philosophy

Emails are sent:

- On **successful completion**
- After **critical state changes**
- When reboots are scheduled or completed

Failures are surfaced via:
- Playbook exit status
- Console logs
- Semaphore history

Email is **not** used as a failure-only channel to avoid noise.

---

## Security Boundaries

The email system enforces:

- No inbound email triggers
- No webhook execution
- No external command execution
- No remote SMTP access

It is strictly **one-way outbound communication**.

---

## Why This Design Was Chosen

This architecture avoids:

- Alert fatigue
- Distributed credentials
- Host-level SMTP exposure
- Silent infrastructure changes

It reinforces the principle that **automation must be observable**.

---

## Summary

The automation email system is:

- Centralized
- Minimal
- Explicit
- Secure
- Human-focused

It provides **confidence**, not noise.

---

## Related Pages

- Ansible Control Node Bootstrap & Trust Model
- Patch & Reboot Orchestration Strategy
- Proxmox VE Patching
- Proxmox Backup Server Automation
- Certificate Issuance & Deployment

---
