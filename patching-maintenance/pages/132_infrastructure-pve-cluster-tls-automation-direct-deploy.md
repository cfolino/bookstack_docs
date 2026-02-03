**Direct TLS Orchestration:** This system manages the automated lifecycle of certificates for the Proxmox hosts (`proxmox`, `pve2`, `pve3`). These nodes **do not terminate on NPM**; certificates are pushed directly to the host filesystem for native termination.

---

# 1. PVE TLS Architecture Overview

* **Target Hosts:** `<internal-host>`, `<internal-host>`, `<internal-host>`
* **CA Authority:** VM 111 (Internal CA)
* **Automation Hub:** VM 103 (Ansible / Semaphore)
* **Dedicated Playbook:** `pve_tls_automation.yml`
* **Termination Path:** `/etc/pve/nodes/<node>/pveproxy-ssl.pem`

---

# 2. PVE TLS Preflight (Mandatory Setup)

Because PVE hosts bypass the Proxy Manager, the identity mapping is handled directly within the PVE Ansible group variables.

### Preflight Process Flow

| Step | Action | Description |
| :--- | :--- | :--- |
| **1** | **DNS Mapping** | Create `A` records in Pi-hole pointing `<internal-host>` to the **Host IP** (not NPM). |
| **2** | **Host Inventory** | Ensure the host exists in `inventory/hosts` under the `[pve]` group. |
| **3** | **Host Vars** | Define `vmid` (if applicable) and host-specific variables in `host_vars/<node>.yml`. |
| **4** | **SAN Definition** | Verify the Management IP is listed as a Subject Alternative Name in `group_vars/pve.yml`. |
| **5** | **SSH Trust** | Confirm VM 103 has root SSH access to the PVE node for certificate deployment. |

---

# 3. PVE TLS Maintenance & Operations

This automation uses a dedicated playbook to avoid conflicts with application-layer certificates.

### Authoritative Commands

| Task | Command |
| :--- | :--- |
| **PVE Cert Renewal** | `ansible-playbook playbooks/pve_tls_automation.yml` |
| **Force Renew** | `ansible-playbook playbooks/pve_tls_automation.yml -e force_renewal=true` |
| **Verify Cert** | `openssl x509 -in /etc/pve/local/pveproxy-ssl.pem -text -noout` |

---

# 4. Monthly Rotation Logic (The 10th)

PVE certificate rotation is bundled with the **Core Infrastructure Window (04:15)** on the 10th of every month.

### Process Flow: Direct Deployment
1. **CSR Generation:** Ansible generates a unique CSR for each PVE node.
2. **CA Signing:** VM 111 (CA) signs the request using the internal root.
3. **Direct Push:** Ansible pushes the `.pem` and `.key` files directly to `/etc/pve/nodes/<node>/`.
4. **Service Reload:** The `pveproxy` service is restarted on the host to apply the new certificate.

---

## Troubleshooting & Safety Gates

* **NPM Bypass:** If `pveproxy` is unreachable, check local host logs. NPM is not involved in this path.
* **Certificate Mismatch:** Ensure the hostname in `/etc/hostname` matches the FQDN in the inventory to prevent SSL handshake errors.
* **Service Lock:** The playbook is designed to restart `pveproxy` only. It does not reboot the entire host unless the monthly patching playbook is also triggered.

### Manual Verification
After running automation, verify the web UI reflects the new expiry date:
```bash
# Run from your workstation
curl -vI [https://internal.example](https://internal.example) 2>&1 | grep "expire date"
