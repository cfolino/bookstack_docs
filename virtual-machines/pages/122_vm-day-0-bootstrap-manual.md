## Purpose
Day-0 bootstrap establishes **identity and intent** for a newly cloned VM.
These steps are performed **once per VM** and are intentionally manual.

Automation begins at Day-1.

---

## Preconditions
- VM cloned from `debian-13-golden`
- VM powered on
- Network reachable (DHCP or temporary IP)

---

## Step 1 — Clone the VM (Proxmox)

Clone from template:
- Template: `debian-13-golden`
- Assign VMID
- Assign name (temporary or final)

Start the VM.

---

## Step 2 — Initial SSH Access (Break-Glass User)

```bash
ssh cfolino@<VM_IP>
```

Verify:
```bash
whoami
hostname
cat /etc/debian_version
```

---

## Step 3 — Set Hostname (Identity)

```bash
sudo hostnamectl set-hostname <new-hostname>
```

Update `/etc/hosts` to resolve the new hostname to localhost (crucial for `sudo` speed resolution on Debian):

```bash
sudo sed -i "s/127.0.1.1.*/127.0.1.1\t<new-hostname>.cfolino.com\t<new-hostname>/" /etc/hosts
```

Apply immediately:
```bash
exec bash
hostname
```

---

## Step 4 — Networking Configuration (Day-0)

This step establishes the VM’s **network identity**.
It is intentionally performed **manually** during Day-0 bootstrap.

Networking changes are **not automated** at this stage to avoid:
- IP conflicts
- VLAN misplacement
- Accidental loss of connectivity

### Step 4.1 — Proxmox Network Placement (Host Side)

Before configuring the guest OS, ensure the VM is attached to the **correct Proxmox bridge**.

Examples:
- `tag=15` → backup network
- `tag=30` → server / internal network (example for this VM)

#### Verify current bridge
```bash
qm config <VMID> | grep net0
```

#### Update bridge if required
```bash
qm set <VMID> --net0 virtio,bridge=vmbr0,tag=30
```

If the bridge is changed:
- Power-cycle the VM (**stop → start**, not reboot)
- Do **not** proceed until the VM boots cleanly on the correct bridge.

### Step 4.2 — Guest Network Verification (Initial, DHCP)

SSH into the VM and verify the current state **before making changes**:

```bash
ip -br a
ip r
systemctl status networking
```

Confirm:
- Correct interface name (likely `ens18` or `enp6s18` depending on standard/virtio).
- Link is **UP**.
- A temporary/DHCP address is present.
- Correct gateway is reachable.

> **Warning:** Do not configure a static IP until link and routing are confirmed working via DHCP.

### Step 4.3 — Disable Cloud-Init Networking (Required)

Debian cloud images often use `cloud-init` to generate `/etc/network/interfaces.d/50-cloud-init`. If you modify networking manually without disabling this, `cloud-init` may overwrite your changes on the next boot or cause conflicts.

#### 1. Disable Cloud-Init Network Configuration
Create a configuration file to tell cloud-init to stop managing networking:

```bash
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

#### 2. Remove Generated Configs
Remove the existing auto-generated file:

```bash
sudo rm -f /etc/network/interfaces.d/50-cloud-init
```

### Step 4.4 — Configure Static IP (Interfaces)

Debian 13 uses the standard `/etc/network/interfaces` file.

1. **Backup the current config:**
   ```bash
   sudo cp /etc/network/interfaces /etc/network/interfaces.bak
   ```

2. **Edit the configuration:**
   ```bash
   sudo nano /etc/network/interfaces
   ```

3. **Define the static configuration.**
   Replace the `dhcp` line for your primary interface (e.g., `ens18`) with the following block.
   *(Example: Setting IP 192.168.x.x on VLAN 30)*

   ```auto
   # The loopback network interface
   auto lo
   iface lo inet loopback

   # The primary network interface
   allow-hotplug ens18
   iface ens18 inet static
       address 192.168.x.x/24
       gateway 192.168.x.x
       # DNS is handled by /etc/resolv.conf, but if 'resolvconf' package is installed:
       # dns-nameservers 192.168.x.x 192.168.x.x
   ```

4. **Update DNS (Resolv.conf)**
   If not using `resolvconf` or `systemd-resolved`, manually set your nameservers:

   ```bash
   sudo nano /etc/resolv.conf
   ```
   ```auto
   nameserver 192.168.x.x
   nameserver 192.168.x.x
   search cfolino.com
   ```

5. **Apply Changes:**
   ```bash
   sudo systemctl restart networking
   ```

   *Troubleshooting Note: If you lose access, use the Proxmox Console.*

---

## Step 5 — Register in Ansible Inventory

Add to your control node inventory (e.g., `hosts.ini` or YAML):

```ini
[web]
homepage ansible_host=192.168.x.x

[internal]
homepage
```

At this point, **manual work is complete**.

Proceed to Day-1 automation.
