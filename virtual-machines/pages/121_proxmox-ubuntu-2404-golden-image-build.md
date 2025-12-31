## Overview
This document describes the creation of a **Proxmox VM golden image** based on **Ubuntu Server 24.04 LTS**, installed from ISO (non–cloud-init), hardened, cleaned, and converted into a reusable template.

This image is intended for:
- Manual + automated VM provisioning
- Ansible/Semaphore-managed environments
- Internal PKI / SSH key-based access
- Predictable, repeatable VM cloning

---

## VM Details

| Item | Value |
|---|---|
VM ID | 9000 |
Name | ubuntu-24.04-golden |
OS | Ubuntu Server 24.04 LTS |
Install Method | ISO installer (no cloud-init) |
Primary User | `cfolino` |
Sudo | Passwordless |
SSH | Keys + password (break-glass) |
Root SSH | Disabled |
Guest Agent | Enabled + verified |

---

## Step 1 — Base Installation (ISO)

- Ubuntu Server 24.04 installed from ISO
- Standard guided storage
- OpenSSH server enabled during install
- User `cfolino` created
- ISO detached after install
- VM rebooted successfully

Verification:
```bash
cat /etc/os-release
whoami
ip -br a
```

---

## Step 2 — Passwordless Sudo

Create a sudoers drop-in:
```bash
sudo -i
echo "cfolino ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/90-cfolino
chmod 440 /etc/sudoers.d/90-cfolino
visudo -cf /etc/sudoers
exit
```

Verify:
```bash
sudo whoami
```

Expected output:
```
root
```

---

## Step 3 — SSH Access

### Switch to SSH
```bash
ssh cfolino@<VM_IP>
```

Verify:
```bash
whoami
sudo whoami
```

---

## Step 4 — QEMU Guest Agent (Critical)

### Install inside VM
```bash
sudo apt update
sudo apt install -y qemu-guest-agent
```

### Enable agent in Proxmox (host)
```bash
qm set 9000 --agent enabled=1
qm stop 9000
qm start 9000
```

### Verify inside VM
```bash
ls -l /dev/virtio-ports/
systemctl status qemu-guest-agent --no-pager
```

Expected:
```
org.qemu.guest_agent.0
Active: active (running)
```

### Verify from Proxmox host
```bash
qm agent 9000 ping
```

---

## Step 5 — SSH Key Installation

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Verify key-based SSH from workstation:
```bash
ssh cfolino@<VM_IP>
```

---

## Step 6 — SSH Hardening (Break-Glass Preserved)

`/etc/ssh/sshd_config`:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
MaxAuthTries 3
LoginGraceTime 30
```

Validate + reload:
```bash
sudo sshd -t
sudo systemctl reload ssh
```

Verify:
```bash
ssh root@<VM_IP>      # denied
ssh cfolino@<VM_IP>   # allowed
```

---

## Step 7 — Baseline System Tweaks

### Enable SSD TRIM
```bash
sudo systemctl enable --now fstrim.timer
```

### Journald limits
`/etc/systemd/journald.conf`:
```ini
SystemMaxUse=200M
RuntimeMaxUse=100M
```

Apply:
```bash
sudo systemctl restart systemd-journald
journalctl --disk-usage
```

---

## Step 8 — MOTD Cleanup

```bash
sudo chmod -x /etc/update-motd.d/10-help-text
sudo chmod -x /etc/update-motd.d/50-motd-news
```

---

## Step 9 — Baseline Tooling (No Vim)

```bash
sudo apt install -y \
  curl wget git htop \
  ca-certificates gnupg \
  net-tools tcpdump \
  chrony
```

Verify time sync:
```bash
systemctl status chrony --no-pager
chronyc tracking
```

---

## Step 10 — Disable Unattended Upgrades

```bash
sudo systemctl disable --now unattended-upgrades
systemctl status unattended-upgrades --no-pager
```

Rationale:
- Updates handled centrally via Ansible/Semaphore
- Prevents surprise reboots and package locks

---

## Step 11 — Remove cloud-init

```bash
sudo apt purge -y cloud-init
sudo rm -rf /etc/cloud
dpkg -l | grep cloud-init || echo "cloud-init removed"
```

---

## Step 12 — Template Hygiene (Critical)

```bash
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```

Ensures unique identity for all clones.

---

## Step 13 — Power Off

```bash
sudo shutdown -h now
```

---

## Step 14 — Convert to Template (Proxmox)

```bash
qm template 9000
```

---

## Final State Summary

| Component | Status |
|---|---|
Guest Agent | Enabled + running |
SSH Keys | Enabled |
Password SSH | Enabled (break-glass) |
Root SSH | Disabled |
cloud-init | Removed |
Unattended Upgrades | Disabled |
machine-id | Reset |
Template | Created |

---

## Notes for Clones

- New machine-id generated on first boot
- SSH keys inherited (remove or replace as needed)
- Password auth available for recovery
- Ready for Ansible bootstrap immediately

---

**Status:** Golden Image Complete
**Template:** `ubuntu-24.04-golden`
