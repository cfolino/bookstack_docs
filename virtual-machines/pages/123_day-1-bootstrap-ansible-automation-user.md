## Purpose
This runbook defines the **Day-1 bootstrap procedure** for a newly deployed VM.
Its sole purpose is to establish **secure, key-based Ansible automation access**.

This procedure is:
- Executed **once per host**
- Performed **after Day-0 OS install**
- Required **before any hardening or configuration roles**

---

## Scope (What Day-1 Does)
- Validates inventory and vault access
- Establishes SSH trust
- Uses the Day-0 break-glass admin to bootstrap automation
- Creates the `ansible` user
- Installs SSH key for automation
- Grants passwordless sudo
- Verifies end-state access

---

## Out of Scope (Explicitly Not Done)
- OS hardening
- SSH lockdown
- Package baselines
- Monitoring
- Certificates
- Firewall rules

These occur **after Day-1**.

---

## Preconditions
- Day-0 OS install complete
- Hostname set
- Networking functional
- Break-glass admin account exists
- Inventory updated
- Ansible Vault configured
- Ansible controller reachable

---

## Step 1 — Inventory Validation (Controller)

```bash
ansible-inventory -i inventory/hosts.yml --list | grep -A5 home
```

**Verify**
- No warnings or errors
- Host appears correctly

**Stop if failed.**

---

## Step 2 — SSH Trust Reset (Controller)

If host was rebuilt:

```bash
ssh-keygen -f "/home/ansible/.ssh/known_hosts" -R "<internal-host>"
```

---

## Step 3 — Break-Glass SSH Login

```bash
ssh cfolino@<internal-host>
```

Password prompt is **expected**.

---

## Step 4 — Verify sudo (Host)

```bash
sudo -n true && echo "sudo: OK" || echo "sudo: FAILED"
```

**Expected**
```text
sudo: OK
```

**Stop if failed.**

---

## Step 5 — Create Bootstrap Playbook (Controller)

File:
```text
playbooks/bootstrap_ansible_user.yml
```

```yaml
---
- name: Bootstrap ansible automation user
  hosts: home
  become: true
  tasks:

    - name: Ensure ansible user exists
      ansible.builtin.user:
        name: ansible
        shell: /bin/bash
        groups: sudo
        append: true
        create_home: true

    - name: Install SSH key for ansible user
      ansible.builtin.authorized_key:
        user: ansible
        state: present
        key: "{{ lookup('file', '/home/ansible/.ssh/id_ansible.pub') }}"

    - name: Allow ansible passwordless sudo
      ansible.builtin.copy:
        dest: /etc/sudoers.d/ansible
        content: "ansible ALL=(ALL) NOPASSWD:ALL\n"
        owner: root
        group: root
        mode: '0440'
```

---

## Step 6 — Syntax Check

```bash
ansible-playbook --syntax-check playbooks/bootstrap_ansible_user.yml
```

---

## Step 7 — Dry Run (Expected Partial Failure)

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  -u cfolino \
  --ask-pass \
  --ask-become-pass \
  playbooks/bootstrap_ansible_user.yml \
  --check
```

**Note**
- Failure on `authorized_key` in check mode is **expected**
- Do not modify playbook

---

## Step 8 — Live Run

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  -u cfolino \
  --ask-pass \
  --ask-become-pass \
  playbooks/bootstrap_ansible_user.yml
```

**Verify**
- No failures
- User created
- SSH key installed
- Sudoers file written

---

## Step 9 — Verify Key-Based SSH

```bash
ssh ansible@<internal-host>
```

**Expected**
- No password prompt
- Immediate login

---

## Step 10 — Verify Passwordless sudo

```bash
sudo -n true && echo "sudo: OK" || echo "sudo: FAILED"
```

**Expected**
```text
sudo: OK
```

---

## Day-1 Exit Criteria (ALL MUST PASS)

- Inventory parses cleanly
- Vault decrypts successfully
- SSH trust established
- `ansible` user exists
- SSH key authentication works
- Passwordless sudo confirmed
- No password flags required for automation

---

## Post-Day-1 Actions
- Set `ansible_user: ansible` in inventory
- Remove all `--ask-pass` usage
- Proceed to Day-2 roles (hardening, baseline, certs, monitoring)

---

## Day-1 Status
**COMPLETE**
