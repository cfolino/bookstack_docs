# npm nGp

* Role: Reverse Proxy and Certificate TerminationPlatform: Ubuntu 24.04 Server (Docker-based Nginx Proxy Manager)Hostname: npm.cfolino.comManagement URL: https://npm.cfolino.comManagement IP: 192.168.10.100VLAN: 1 (Infrastructure LAN)
* Notes:Static IP assigned via NetplanDNS: 192.168.10.44, 192.168.10.46Gateway: 192.168.10.1Proxmox VMID: 105Resources: 2 vCPU / 4 GB RAM / 32 GB disk
* Access:Internal-only access; external access restricted by firewall policies
* Usage:Terminates HTTPS for internal applicationsForwards requests to backend VMs and containersStores and serves CA-signed certificates for all internal FQDNsProvides centralized proxy host management and TLS orchestration
