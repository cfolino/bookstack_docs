## Purpose
Day-0 bootstrap establishes **identity and intent** for a newly cloned VM.
These steps are performed **once per VM** and are intentionally manual.

Automation begins at Day-1.

---

## Preconditions
- VM cloned from `ubuntu-24.04-golden`
- VM powered on
- Network reachable (DHCP or temporary IP)

---

## Step 1 — Clone the VM (Proxmox)

Clone from template:
- Template: `ubuntu-24.04-golden`
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
```

---

## Step 3 — Set Hostname (Identity)

```bash
sudo hostnamectl set-hostname <internal-host>
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

---

### Step 4.1 — Proxmox Network Placement (Host Side)

Before configuring the guest OS, ensure the VM is attached to the **correct Proxmox bridge**.

Examples:
- `vmbr0`  → default / management
- `vmbr15` → backup network
- `vmbr30` → server / internal network (example for this VM)

#### Verify current bridge
```bash
qm config <VMID> | grep net0
```

#### Update bridge if required
```bash
qm set <VMID> --net0 virtio,bridge=vmbr30
```

If the bridge is changed:
- Power-cycle the VM (**stop → start**, not reboot)
- Do **not** proceed until the VM boots cleanly on the correct bridge

---

### Step 4.2 — Guest Network Verification (Initial, DHCP)

SSH into the VM and verify the current state **before making changes**:

```bash
ip -br a
ip r
resolvectl status
```

Confirm:
- Correct interface name (e.g. `enp6s18`)
- Link is **UP**
- A temporary/DHCP address is present
- Correct gateway is reachable

⚠️ **Do not configure a static IP until link, routing, and DNS are confirmed working via DHCP.**

---

### Step 4.3 — Disable cloud-init Netplan (Required)

Ubuntu installations may include a cloud-init generated netplan file that enables DHCP.
If left in place, this can cause **multiple IP addresses** (DHCP + static) on the same interface.

#### Check for cloud-init netplan files
```bash
ls -l /etc/netplan
```

If `50-cloud-init.yaml` exists, disable it:

```bash
sudo mkdir -p /etc/netplan/disabled
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/disabled/
```

Verify only one active netplan file remains:
```bash
ls -l /etc/netplan
```

---

### Step 4.4 — Configure Static IP (Netplan)

Create or edit the **single authoritative** netplan file:

```bash
sudo nano /etc/netplan
```
---

## Step 5 — Register in Ansible Inventory

Add to inventory (example):

```ini
[web]
homepage ansible_host=192.168.x.x

[internal]
homepage
```

At this point, **manual work is complete**.

Proceed to Day-1 automation.
