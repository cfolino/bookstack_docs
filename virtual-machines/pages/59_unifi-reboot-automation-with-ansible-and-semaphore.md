Version: v1.0
Scope: Centralized UniFi device + UDM Pro reboot automation via Ansible, driven by Semaphore.
Environment: ansible.cfolino.com, semaphore.cfolino.com, udm.cfolino.com

---

## 1. Overview

This document explains the complete configuration, execution flow, and verification procedures for automating UniFi device and UDM Pro reboots using:

- Ansible playbooks
- The `unifi_reboot` role
- Inventory-driven variable management
- Semaphore as an execution and scheduling engine
- Notification delivery using the `notify_email` role

All dry-run logic has been removed from the workflow. A dry-run may still be performed manually using Ansible `--check` mode.

---

## 2. Repository Structure

```
ansible/
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       ├── unifi.yml   (merged into all.yml)
│       └── smtp.yml    (merged into all.yml)
├── playbooks/
│   ├── reboot_unifi.yml
│   ├── notify_test.yml
│   └── unifi_api_test.yml
└── roles/
    ├── unifi_reboot/
    │   └── tasks/
    │       ├── reboot_api.yml
    │       ├── reboot_api_per_device.yml
    │       ├── reboot_device.yml
    │       ├── reboot_udm.yml
    │       ├── summary.yml
    │       └── main.yml
    └── notify_email/
        └── tasks/main.yml
```

---

## 3. Inventory Configuration

`localhost` must exist in the inventory so Ansible automatically loads `group_vars/all.yml`.

Location: `inventory/hosts.yml`

```
all:
  children:

    infrastructure:
      hosts:
        pve:
        pbs:
        ansible:
        omv:

    proxy:
      hosts:
        npm:

    dns:
      hosts:
        pihole:
        pihole-backup:

    certificate_authority:
      hosts:
        ca:

    internal:
      hosts:
        bookstack:
        grafana:

    unifi:
      hosts:
        udmpro:

  hosts:

    localhost:
      ansible_connection: local

    pve:
      ansible_host: pve.internal.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    pbs:
      ansible_host: pbs.internal.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    ansible:
      ansible_host: ansible.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    pihole:
      ansible_host: pihole.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    pihole-backup:
      ansible_host: pihole-backup.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    ca:
      ansible_host: ca.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    bookstack:
      ansible_host: bookstack.internal.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    omv:
      ansible_host: omv.internal.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    grafana:
      ansible_host: grafana.internal.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    npm:
      ansible_host: npm.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible

    udmpro:
      ansible_host: udm.cfolino.com
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible
```

---

## 4. Variable Management

All operational variables live in:

```
inventory/group_vars/all.yml
```

Example:

```
# UniFi API configuration
udm_host: "https://udm.cfolino.com"
udm_api_key: "{{ vault_udm_api_key }}"
udm_ssh_host: "udm.cfolino.com"
udm_ssh_user: "ansible"

# SMTP configuration
smtp_host: "smtp.gmail.com"
smtp_port: 587
smtp_user: "ansible@cfolino.com"
smtp_secure: "starttls"
notify_to: "you@cfolino.com"
smtp_pass: "{{ vault_smtp_pass }}"
```

Sensitive values are injected via Semaphore variable group.

---

## 5. Semaphore Template Configuration

Open:

```
Task Templates > UniFi Devices Reboot
```

Settings:

```
Name: UniFi Devices Reboot
Type: Playbook
Repository: local
Branch: main
Inventory: main
Playbook Path: playbooks/reboot_unifi.yml
Variable Group: unifi_reboot_secrets
```

### Environment Variables

```
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

### Semaphore Variable Group (secrets only)

```
vault_udm_api_key: "REDACTED"
vault_smtp_pass: "REDACTED"
```

---

## 6. Playbook Configuration

Location:
```
playbooks/reboot_unifi.yml
```

```
---
- name: Reboot UniFi devices (API + UDM SSH)
  hosts: localhost
  gather_facts: no

  environment:
    REQUESTS_CA_BUNDLE: "/etc/ssl/certs/ca-certificates.crt"

  roles:
    - role: unifi_reboot
      tags: unifi
```

---

## 7. Execution Flow

1. Query UniFi API for device list
2. Filter active non-UDM devices
3. Reboot each device via UniFi API
4. Reboot UDM Pro via SSH
5. Wait for UDM SSH port to return
6. Generate HTML summary
7. Send email notification

---

## 8. Pre-Flight Testing Procedures

### 8.1 API Connectivity Test

```
ansible localhost -m uri -a \
"url=https://udm.cfolino.com/proxy/network/api/s/default/stat/device \
method=GET validate_certs=true \
headers='{\"X-API-Key\":\"<KEY>\"}'" -vvv
```

### 8.2 SSH Connectivity Test

```
ssh ansible@udm.cfolino.com "echo ok"
```

### 8.3 SMTP Test

```
ansible-playbook playbooks/notify_test.yml
```

---

## 9. Running the Automation

### 9.1 Semaphore

1. Open the template
2. Create Task
3. Leave "Extra CLI Args" empty
4. Execute
5. Review output
6. Verify email summary

### 9.2 Manual CLI Run

```
ansible-playbook playbooks/reboot_unifi.yml -vv
```

---

## 10. Expected Runtime Output Sequence

1. Reboot UniFi devices via API
2. Reboot UDM Pro via SSH
3. Wait for UDM SSH on port 22
4. Build and email summary

---

## 11. Troubleshooting

### 11.1 Undefined json in API filter

Cause:
- Running with `--check`
- API unreachable

Fix:
- Run without check mode
- Validate API connectivity

### 11.2 Certificate issues

Ensure:

```
REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
```

### 11.3 Missing variables

```
ansible localhost -m debug -a "var=hostvars['localhost']"
```

---

## 12. Summary

This automation provides a robust, inventory-driven reboot workflow for UniFi devices and the UDM Pro. Configuration is governed by Ansible `group_vars`, secrets are managed by Semaphore, and execution is controlled by a single centralized template. All tasks conclude with an HTML email summary.
