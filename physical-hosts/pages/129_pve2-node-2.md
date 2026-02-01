**Proxmox Node 2:** This node serves as a secondary hypervisor and replication target in the 3-node cluster.

# Proxmox Node 2 Overview

* **Role:** Secondary Hypervisor / Replication Target
* **Hypervisor Platform:** Proxmox VE
* **Hostname:** `<internal-host>`
* **Management IP:** `192.168.x.x`
* **VLAN:** 1
* **Hardware:** Dell OptiPlex Micro (MFF)

---

## Network Configuration

This host utilizes a VLAN-aware bridge to support seamless VM migration and High Availability (HA) failover.

| Interface | IP Address | Purpose |
| :--- | :--- | :--- |
| `vmbr0` | `192.168.x.x` | Management & VM Traffic (VLAN Aware) |
| `vmbr0.15` | `192.168.x.x` | Dedicated Replication & Migration (VLAN 15) |

### Process: Applying Network Parity
If the configuration drifts, ensure these lines are present in `/etc/network/interfaces` under `vmbr0`:

```bash
# bridge-vlan-aware yes
# bridge-vlan-protocol 802.1q

ifreload -a
