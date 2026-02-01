## Overview
This document describes the creation of a **Proxmox VM golden image** based on **Debian 12 (Bookworm)**. It is hardened, stripped of unique identifiers, and optimized for the `cfolino.com` infrastructure, ensuring seamless integration with Ansible and Semaphore.

---

## VM Details

| Item | Value |
| :--- | :--- |
| **VM ID** | 9001 |
| **Name** | debian-12-golden |
| **OS** | Debian 12 (Bookworm) |
| **Install Method** | ISO Netinst (`debian-12.x.x-amd64-netinst.iso`) |
| **Primary User** | `cfolino` |
| **Sudo** | Passwordless |
| **SSH** | Keys + password (break-glass) |
| **Root SSH** | Disabled |
| **Guest Agent** | Enabled + verified |

---

## Step 1 — Base Installation (ISO)

1. **Software Selection:** - [ ] Debian desktop environment
   - [ ] GNOME
   - [x] SSH server
   - [x] standard system utilities
2. **User Creation:** Create user `cfolino`.
3. **Root Password:** Leave **blank**. This automatically installs `sudo` and grants the first user (`cfolino`) administrative rights.
4. **Partitioning:** Guided - use entire disk (Ext4).

---

## Step 2 — Passwordless Sudo

Configure the `cfolino` user for Ansible-friendly passwordless execution:

```bash
sudo -i
echo "cfolino ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-cfolino
chmod 440 /etc/sudoers.d/90-cfolino
visudo -cf /etc/sudoers
exit
```

---

## Step 3 — QEMU Guest Agent

Install the agent and enable the service for Proxmox communication.

```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

**Verification (from Proxmox Host):**
```bash
qm set 9001 --agent enabled=1
qm agent 9001 ping
```

---

## Step 4 — SSH Configuration & Hardening

### 1. Inject Management Public Key
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
# Add the management/Ansible public key
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 2. Hardening `/etc/ssh/sshd_config`
```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes
MaxAuthTries 3
LoginGraceTime 30
```

### 3. Reload Service
```bash
sudo sshd -t
sudo systemctl reload ssh
```

---

## Step 5 — Baseline System Tweaks

### SSD TRIM
```bash
sudo systemctl enable --now fstrim.timer
```

### Journald Limits
Edit `/etc/systemd/journald.conf`:
```ini
SystemMaxUse=200M
RuntimeMaxUse=100M
```
```bash
sudo systemctl restart systemd-journald
```

---

## Step 6 — Baseline Tooling (Standard Fleet)

Install necessary packages for `cfolino.com` observability and management:

```bash
sudo apt install -y \
  curl wget git htop \
  ca-certificates gnupg \
  net-tools tcpdump \
  chrony rsync
```

---

## Step 7 — Disable Unwanted Automation

Ensure Ansible remains the source of truth by removing `cloud-init` (if present) and `unattended-upgrades`.

```bash
sudo apt purge -y cloud-init unattended-upgrades
sudo apt autoremove -y
```

---

## Step 8 — Template Hygiene (Critical)

This ensures that cloned VMs do not suffer from IP conflicts or duplicate IDs in logs.

```bash
# Clean apt cache
sudo apt clean

# Rotate and vacuum logs
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

# Reset Machine ID
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Clear shell history and power off
history -c && sudo shutdown -h now
```

---

## Step 9 — Convert to Template

On the Proxmox Host:
```bash
qm template 9001
```

---

## Final State Summary

| Component | Status |
| :--- | :--- |
| **Guest Agent** | Enabled + running |
| **SSH Keys** | Authorized for `cfolino` |
| **Password SSH** | Enabled (Break-glass) |
| **Root SSH** | Disabled |
| **CA Trust** | Ready for internal CA |
| **Machine ID** | Truncated (Unique on boot) |

**Status:** Golden Image Complete
**Template:** `debian-12-golden`
