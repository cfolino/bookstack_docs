### Purpose

Automates the creation of the `ansible` user on new virtual machines, sets up secure SSH key-based authentication, and enforces hardened SSH access.

---

### Overview

* Creates the `ansible` user with `/bin/bash` shell (if not already present).
* Sets up `.ssh` directory and installs public key from the Ansible control node.
* **Disables password authentication** for the `ansible` user at the SSH daemon level.
* Reloads the SSH service to apply changes.

---

### Prerequisites

* Run as `root` or with `sudo` privileges.
* SSH public key from control node must be available and correct.
* SSH server must be active and properly configured.

---

### Script Content /home/ansible/ansible/ansible-user-init.sh

```bash
#!/bin/bash

ANSIBLE_USER="ansible"
SSH_DIR="/home/${ANSIBLE_USER}/.ssh"
AUTHORIZED_KEYS="${SSH_DIR}/authorized_keys"

# Public key from your Ansible control node
PUB_KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINCTCTvLEv+uBz6gCPsA2UOcmyYTavT1O556ZIMuz5jR ansible@cfolino.com"

# Create ansible user if not already present
if id "${ANSIBLE_USER}" &>/dev/null; then
    echo "User ${ANSIBLE_USER} already exists"
else
    sudo useradd -m -s /bin/bash "${ANSIBLE_USER}"
    echo "User ${ANSIBLE_USER} created"
fi

# Prepare .ssh directory
sudo mkdir -p "${SSH_DIR}"
sudo chown "${ANSIBLE_USER}:${ANSIBLE_USER}" "${SSH_DIR}"
sudo chmod 700 "${SSH_DIR}"

# Add public key
echo "${PUB_KEY}" | sudo tee "${AUTHORIZED_KEYS}" > /dev/null
sudo chown "${ANSIBLE_USER}:${ANSIBLE_USER}" "${AUTHORIZED_KEYS}"
sudo chmod 600 "${AUTHORIZED_KEYS}"

# Harden SSH: disable password authentication for ansible user
echo -e "Match User ${ANSIBLE_USER}\n    PasswordAuthentication no" | \
    sudo tee /etc/ssh/sshd_config.d/99-ansible-user.conf > /dev/null
sudo systemctl reload ssh

echo "SSH key deployed and password login disabled. 'ansible' user is ready for key-based login."
```

---

### Usage

Run the script on each new VM after provisioning:

```bash
sudo bash ansible-user-init.sh
```

---

### Hardening Summary

| Step                    | Action                                                            |
| ----------------------- | ----------------------------------------------------------------- |
| SSH Directory           | `.ssh/` created with 700 permissions                              |
| Authorized Key          | Installed with 600 permissions and correct ownership              |
| Password Authentication | Explicitly disabled for `ansible` user using SSHD match directive |
| SSH Reload              | `systemctl reload ssh` applied for changes to take effect         |

---

### Validation

From your Ansible control node, run:

```bash
ssh -i ~/.ssh/id_ansible ansible@<target-vm-ip>
```

To verify:

* Login is successful using key
* `ansible` user **cannot** log in using a password (even if one is set)

---

### Notes

* The config is written to `/etc/ssh/sshd_config.d/99-ansible-user.conf` to keep it modular.
* Applies safely alongside any existing SSH config and avoids modifying the main `sshd_config` file.
* This can be extended with other `Match User` restrictions (e.g. `AllowUsers`, `PermitTTY no`).

---

## Workflow Summary

| Step                 | Description                                    |
| -------------------- | ---------------------------------------------- |
| 1. User creation     | Creates `ansible` user with `/bin/bash` shell  |
| 2. SSH key setup     | Installs control node public key securely      |
| 3. SSH hardening     | Disables password login for `ansible` only     |
| 4. SSH daemon reload | Reloads to apply new match rule                |
| 5. Ready for Ansible | VM is now secure and accessible for automation |

---
