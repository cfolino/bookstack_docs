# grafana

* Centralized monitoring and visualization platform for homelab infrastructure and services.
* Role: Monitoring and Dashboard PlatformPlatform: Ubuntu 24.04 ServerHostname: grafana.cfolino.comManagement URL: Web UI at http://grafana.cfolino.com:3000Management IP: 192.168.10.13VLAN: 1 (Infrastructure LAN)Notes:Static IP: assigned via Netplan to interface enp6s18DNS: 192.168.10.44, 192.168.10.46Gateway: 192.168.10.1Proxmox VMID: 104Resources: 2 vCPU / 4 GB RAM / 32 GB diskIntegrated with: Prometheus for metrics collectionAccess: Accessible internally only; firewall rules restrict external access
