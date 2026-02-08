## Overview
This document describes the creation of a **Proxmox VM golden image** based on **Debian 13 (Trixie)**. It is hardened, stripped of unique identifiers, and optimized for the `cfolino.com` infrastructure, ensuring seamless integration with Ansible and Semaphore.

---

## VM Details

| Item | Value |
| :--- | :--- |
| **VM ID** | 9002 |
| **Name** | debian-13-golden |
| **OS** | Debian 13 (Trixie) - Testing/Stable |
| **Install Method** | ISO Netinst (`debian-testing-amd64-netinst.iso`) |
| **Primary User** | `cfolino` |
| **Sudo** | Passwordless |
| **SSH** | Keys + password (break-glass) |
| **Root SSH** | Disabled |
| **Guest Agent** | Enabled + verified |

---

## Step 1 — Base Installation (ISO)

1. **Software Selection:**
   - [ ] Debian desktop environment
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

Install the agent and enable the service for Proxmox communication (shutdown/reboot/IP display).

```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

**Verification (from Proxmox Host):**
```bash
qm set 9002 --agent enabled=1
qm agent 9002 ping
```
*(Result should be generic JSON output or empty success, not an error).*

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
Ensure the underlying storage is notified of deleted blocks (crucial for ZFS/Ceph backing).
```bash
sudo systemctl enable --now fstrim.timer
```

### Journald Limits
Prevent logs from filling the small root disk.
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

Install necessary packages for `cfolino.com` observability and management.
*Note: `net-tools` is deprecated but included for legacy `ifconfig`/`netstat` support. Primary networking uses `iproute2`.*

```bash
sudo apt install -y \
  curl wget git htop \
  ca-certificates gnupg \
  net-tools tcpdump \
  chrony rsync
```

---

## Step 7 — Remove Conflicting Automation

We manage Day-0 networking manually. Remove `cloud-init` to prevent it from regenerating network configs or machine-IDs on boot.

```bash
sudo apt purge -y cloud-init cloud-init-ramfs-dyn-netconf unattended-upgrades
sudo apt autoremove -y
# Ensure no residual config files remain
sudo rm -rf /etc/cloud/
sudo rm -rf /var/lib/cloud/
```

---

## Step 8 — Reset Networking to DHCP

Ensure the interface is clean and set to DHCP for the first boot of any clone.
Edit `/etc/network/interfaces`:

```auto
auto lo
iface lo inet loopback

allow-hotplug ens18
iface ens18 inet dhcp
```
*(Adjust `ens18` if your VM uses a different driver, but VirtIO usually maps to `ens18`)*.

---

## Step 9 — Template Hygiene (Critical)

This ensures that cloned VMs do not suffer from IP conflicts or duplicate IDs in logs.

```bash
# Clean apt cache
sudo apt clean

# Rotate and vacuum logs
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s

# Reset Machine ID (Critical for DHCP and Systemd uniqueness)
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Clear shell history and power off
history -c && sudo shutdown -h now
```

---

## Step 10 — Convert to Template

On the Proxmox Host:
```bash
qm template 9002
```

---

## Final State Summary

| Component | Status |
| :--- | :--- |
| **Guest Agent** | Enabled + running |
| **SSH Keys** | Authorized for `cfolino` |
| **Password SSH** | Enabled (Break-glass) |
| **Cloud-Init** | **PURGED** (Manual Day-0 required) |
| **CA Trust** | Ready for internal CA |
| **Machine ID** | Truncated (Unique on boot) |

**Status:** Golden Image Complete
**Template:** `debian-13-golden`
