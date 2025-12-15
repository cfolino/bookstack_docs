* Role: Monitoring and Dashboard Platform
* Platform: Ubuntu 24.04 Server
* Hostname: <internal-host>
* Management URL: Web UI at https://internal.example
* Management IP: 192.168.x.x
* VLAN: 1 (Infrastructure LAN)
* Notes:
* Static IP: assigned via Netplan to interface enp6s18
* DNS: 192.168.x.x, 192.168.x.x
* Gateway: 192.168.x.x
* Proxmox VMID: 104
* Resources: 2 vCPU / 4 GB RAM / 32 GB disk
* Integrated with: Prometheus for metrics collection
* Access: Accessible internally only; firewall rules restrict external access
