---

This page documents the shared notification and email automation used across the homelab’s Ansible workflows. The `notify_email` role provides a consistent, reusable mechanism for sending structured HTML email notifications after significant automation events.

This role is intentionally centralized and is treated as a **shared infrastructure service** within Ansible.

---

## Purpose

The notification system is used to:
- Confirm successful patching and reboots
- Capture operational summaries
- Surface warnings or state changes
- Provide asynchronous visibility into automation runs

Notifications are sent **after state changes**, not before, ensuring emails reflect actual system outcomes.

---

## Role Location

```text
roles/notify_email/
├── defaults/
│   └── main.yml
├── files/
│   └── sendmail_gmail.py
└── tasks/
    └── main.yml
```

---

## Design Overview

The notification workflow consists of:
1. Automation role builds a summary (`set_fact`)
2. Summary content is passed to `notify_email`
3. Email is sent from the Ansible control node
4. HTML-formatted output is delivered via SMTP

The role itself is intentionally **logic-light**. All semantic meaning lives in the calling playbook or role.

---

## Role Interface (Inputs)

The role expects two variables to be defined by the caller:

```yaml
notify_subject: "Descriptive subject line"
notify_body: "<h3>HTML-formatted body</h3>"
```

These variables are constructed dynamically by:
- Patching roles
- Certificate automation
- Infrastructure workflows

---

## Defaults (`defaults/main.yml`)

```yaml
smtp_server: smtp.gmail.com
smtp_port: 587
smtp_secure: starttls
```

Authentication credentials are stored securely and are not hardcoded in the role.

---

## Email Transport Mechanism

Email delivery is handled by a custom Python helper:

```text
roles/notify_email/files/sendmail_gmail.py
```

### Why a Custom Script?
- Precise control over TLS behavior
- Consistent HTML rendering
- Avoids external Ansible dependencies
- Predictable behavior across hosts

The script:
- Establishes a STARTTLS connection
- Authenticates with Gmail SMTP
- Sends a single HTML message
- Exits cleanly with error propagation

---

## Task Flow (`tasks/main.yml`)

High-level execution sequence:

1. Validate required variables are present
2. Execute `sendmail_gmail.py` locally
3. Pass subject/body via stdin/arguments
4. Send email
5. Return status to Ansible

The task always runs on:
```yaml
delegate_to: localhost
```

This guarantees:
- Emails are sent from the control node
- No SMTP access required on target hosts
- No credentials distributed to managed nodes

---

## Example Usage (From a Role)

```yaml
- name: Send patching summary email
  include_role:
    name: notify_email
  vars:
    notify_subject: "{{ ansible_node_notify_subject }}"
    notify_body: "{{ ansible_node_notify_body }}"
```

---

## HTML Formatting Strategy

Emails are sent as **HTML**, not plaintext.

Common formatting patterns:
- `<h3>` section headers
- `<pre>` blocks for command output
- Clear separation of pre- and post-state
- Minimal styling (mail-client compatible)

This allows emails to double as:
- Audit logs
- Operational reports
- Troubleshooting artifacts

---

## Where Notifications Are Used

The `notify_email` role is currently used by:

- Ansible control node patching
- OMV patching and reboot
- PBS patching and service validation
- Proxmox VE patching
- Certificate issuance and deployment workflows
- Test and validation playbooks

This ensures **consistent visibility** across the entire automation surface.

---

## Security Model

- SMTP credentials exist only on the Ansible control node
- No secrets stored in inventory files
- No credentials copied to managed hosts
- TLS enforced via STARTTLS
- Emails sent only from trusted automation context

---

## Testing & Validation

### Test Notification
```bash
ansible-playbook playbooks/notify_test.yml
```

### Manual SMTP Verification
```bash
python3 roles/notify_email/files/sendmail_gmail.py
```

(Used only during development or troubleshooting.)

---

## Failure Behavior

- Email failures do **not** mask automation failures
- SMTP errors are surfaced in Ansible output
- Patch/reboot logic completes before notification attempt
- Notification is informational, not transactional

---

## Design Principles

- Single responsibility
- Centralized transport logic
- Decoupled from business logic
- Safe delegation
- Predictable output

---

## Related Pages

- Ansible Inventory & Group Design
- Patching & Reboot Automation
- Internal Certificate Authority Automation
- Monitoring & Observability

---
