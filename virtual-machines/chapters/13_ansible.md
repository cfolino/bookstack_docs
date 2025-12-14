# ansible

* Centralized automation and orchestration node for homelab infrastructure configuration and deployment.
* Role: Infrastructure Automation PlatformPlatform: Ubuntu 24.04 ServerHostname: ansible.cfolino.comManagement URL: SSH access via ssh ansible@192.168.10.11Management IP: 192.168.10.11Port: 3000
* Notes:- Static IP assigned to interface enp6s18 via Netplan- DNS: 192.168.10.44, 192.168.10.46- Gateway: 192.168.10.1- Ansible user with SSH key-based auth (/home/ansible/.ssh/id_ansible)- Proxmox VMID: 103- VM resources: 2 vCPU / 4 GB RAM / 32 GB disk- Hosts static Ansible inventory and core playbooks- Automates provisioning, patching, monitoring integration, and TLS certificate management
