#

**Proxmox Node 3:** This node serves as a secondary hypervisor and the current host for Ansible/Management services.

# Proxmox Node 3 Overview

* **Role:** Secondary Hypervisor / Replication Target
* **Hypervisor Platform:** Proxmox VE
* **Hostname:** `<internal-host>`
* **Management IP:** `192.168.x.x`
* **VLAN:** 1
* **Hardware:** Dell OptiPlex Micro (MFF)

---

## Network Configuration

| Interface | IP Address | Purpose |
| :--- | :--- | :--- |
| `vmbr0` | `192.168.x.x` | Management & VM Traffic (VLAN Aware) |
| `vmbr0.15` | `192.168.x.x` | Dedicated Replication & Migration (VLAN 15) |

### Post-Migration Recovery
If a VM loses connectivity after moving to this node, force an ARP update from the host shell:

```bash
# Required to update physical switch ARP tables
ping -I vmbr0 <VM_IP>
