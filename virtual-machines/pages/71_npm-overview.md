* Role: Reverse Proxy and Certificate Termination
* Platform: Ubuntu 24.04 Server (Docker-based Nginx Proxy Manager)
* Hostname: <internal-host>
* Management URL: https://internal.example
* Management IP: 192.168.10.100
* VLAN: 1 (Infrastructure LAN)

Notes:
* Static IP assigned via Netplan
* DNS: 192.168.10.44, 192.168.10.46
* Gateway: 192.168.10.1
* Proxmox VMID: 105
* Resources: 2 vCPU / 4 GB RAM / 32 GB disk

Access:
* Internal-only access; external access restricted by firewall policies

Usage:
* Terminates HTTPS for internal applications
* Forwards requests to backend VMs and containers
* Stores and serves CA-signed certificates for all internal FQDNs
* Provides centralized proxy host management and TLS orchestration
