# ca internal certificate authority

* Internal Certificate Authority VM for issuing and managing TLS certificates in the homelab.
* Role: Certificate Authority (CA)Platform: Ubuntu 24.04 ServerHostname: ca.cfolino.comManagement IP: 192.168.10.99VLAN: 1
* Notes:
* Static IP assigned to interface ens18
* Uses bridged networking via vmbr0
* DNS servers: 192.168.10.44, 192.168.10.46
* Gateway: 192.168.10.1
* Internal domain: *.cfolino.com
* Default user: ca
