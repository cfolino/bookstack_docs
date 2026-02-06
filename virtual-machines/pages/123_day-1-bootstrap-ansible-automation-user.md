## Purpose
This runbook defines the **Day-1 bootstrap procedure** for a newly deployed VM cloned from the Debian 13 Golden Image. Its sole purpose is to establish **secure, key-based Ansible automation access**.

---

## Preconditions
* **Template Source:** `debian-13-golden` (VMID 9001).
* **Day-0 Complete:** Hostname set and networking functional.
* **Identity:** `cfolino` account has the `id_cfolino.pub` key and `NOPASSWD: ALL` sudo access pre-baked.
* **Controller:** Local workstation has the `id_cfolino` private key on the `A:` drive.

---

## Step 1 — Inventory & Host Key Management

Because the `machine-id` was reset in the template, every clone generates a new SSH host key. You must clear any old entries for that IP to avoid "Host Key Verification Failed" errors.

```bash
# Clear old keys for the specific IP
ssh-keygen -f "$HOME/.ssh/known_hosts" -R "192.168.10.XX"
```

---

## Step 2 — Verify Bootstrap Access (Break-Glass)

Verify connectivity using your identity file. No password should be requested.

```bash
ssh -i "A:\ssh\id_cfolino" cfolino@192.168.10.XX
```

---

## Step 3 — The Day-1 Bootstrap Playbook

This playbook creates the standard `ansible` user, injects the automation key, and ensures `sudo` is configured correctly for the Debian `visudo` path.

**File:** `playbooks/bootstrap_ansible_user.yml`

```yaml
---
- name: Bootstrap ansible automation user
  hosts: all
  become: true
  tasks:
    - name: Ensure sudo group exists
      ansible.builtin.group:
        name: sudo
        state: present

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
        key: "{{ lookup('file', 'A:/ssh/id_ansible.pub') }}"

    - name: Allow ansible passwordless sudo
      ansible.builtin.copy:
        dest: /etc/sudoers.d/ansible
        content: "ansible ALL=(ALL) NOPASSWD:ALL"
        owner: root
        group: root
        mode: '0440'
        validate: '/usr/sbin/visudo -cf %s'
```

---

## Step 4 — Execute Bootstrap

Run the playbook from your controller. No password flags are needed because of the `90-cfolino` template configuration.

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  -u cfolino \
  --private-key="A:\ssh\id_cfolino" \
  playbooks/bootstrap_ansible_user.yml
```

---

## Step 5 — Verification of Exit Criteria

**Verify Key-Based Login:**
```bash
ssh -i "A:\ssh\id_ansible" ansible@192.168.10.XX
```

**Verify Passwordless Sudo:**
```bash
sudo -n true && echo "sudo: OK" || echo "sudo: FAILED"
```

---

## Post-Day-1 Status: **COMPLETE**
The host is now ready for Day-2 hardening and role deployment.
Update your inventory to use `ansible_user: ansible`.
