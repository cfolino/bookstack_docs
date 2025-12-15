---

## Purpose
Automates the **generation**, **signing**, and **deployment** of SSL certificates from the internal Certificate Authority (CA) to the **Nginx Proxy Manager (NPM)** host using **Ansible**.
Ensures each host’s **FQDN** receives a valid internal certificate and that **NPM automatically restarts** with the new credentials.

---

## Preparation and Verification Workflow

### 1. Add Host to Ansible Inventory
Edit `/home/ansible/ansible/inventory/inventory.yml` and ensure the host exists with its internal hostname:

```
all:
  hosts:
    pbs:
      ansible_host: pbs.internal.internal
      ansible_user: ansible
      ansible_ssh_private_key_file: /home/ansible/.ssh/id_ansible
```

This ensures Ansible communicates with the internal VM directly rather than the proxied address.

### 2. Initialize the ansible User (If New Host)
Run the initialization playbook to create the user and deploy the SSH key pair before continuing.

### 3. Create DNS Records in Pi-hole
In the **Pi-hole GUI** (`Local DNS → DNS Records`):

| Domain | IP | Purpose |
|---------|----|----------|
| `pbs.internal` | `192.168.x.x` | Points to NPM proxy host |
| `pbs.internal.internal` | `192.168.15.x` | Points directly to PBS VM |

Verify DNS resolution:

```bash
dig +short pbs.internal
dig +short pbs.internal.internal
```
### 4. Verify SSH Connectivity (Semaphore)
Run the `ping.yml` playbook in Semaphore to confirm passwordless access.

Expected log snippet:

```
ok: [pbs]
PLAY RECAP *********************************************************************
pbs : ok=1 changed=0 unreachable=0 failed=0
```

### 5. Add Proxy Host and Dummy Certificate in NPM
1. Log in to **https://npm.internal**
2. Go to **Hosts → Proxy Hosts → Add Proxy Host**
3. Configure:
   - Domain: `pbs.internal`
   - Forward Hostname/IP: `pbs.internal.internal`
   - Forward Port: `8007`
   - Block Common Exploits: Yes
   - WebSocket Support: Yes
   - Access List: `internal-only`
   - SSL: *none yet*
4. Save the host.
5. Upload the pre-generated dummy cert and key stored at `A:\ssh`:
   - SSL Certificates → Add SSL Certificate → Custom
   - Upload `dummy.crt` and `dummy.key`
   - Assign it to the new proxy host.

### 6. Determine Certificate ID
After saving, run:

```bash
sudo ls -1t /home/npm/npm/data/custom_ssl/ | head -n 1
```

Example output:

```bash
npm-14
```

### 7. Update Mapping
Edit `/home/ansible/ansible/inventory/host_vars/npm.yml`:

```
npm_cert_map:
  grafana.internal: 3
  npm.internal: 7
  pve.internal: 4
  bookstack.internal: 6
  prometheus.internal: 8
  node-exporter.internal: 9
  pihole.internal: 10
  semaphore.internal: 11
  omv.internal: 13
  pbs.internal: 14
```

### 8. Open Firewall Between NPM and Target Host
Allow NPM (192.168.x.x) to reach PBS (192.168.15.x) over the service port.

| Field | Value |
|--------|-------|
| Action | Accept |
| Description | Allow NPM to reach PBS |
| Source | 192.168.x.x |
| Destination | 192.168.15.x |
| Protocol | TCP |
| Port | 8007 |

Confirm connectivity:

```bash
curl -Ik https://pbs.internal.internal:8007
```

### 9. Run Certificate Playbook
Execute from the Ansible control node:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/issue_and_deploy_cert.yml
```

### 10. Verify Certificate on NPM Host
Check certificate details:

```bash
openssl x509 -in /home/npm/npm/data/custom_ssl/npm-14/fullchain.pem -noout -subject -issuer -dates
```

Expected output:

```
subject=O = cfolino, OU = homelab, CN = pbs.internal
issuer=C = US, O = Homelab, CN = cfolino Root CA
```

### 11. Apply the Signed Certificate in NPM
Edit the proxy host entry for `pbs.internal` and apply:

| Setting | Value |
|----------|--------|
| Scheme | HTTPS |
| Forward Hostname/IP | pbs.internal.internal |
| Forward Port | 8007 |
| Block Common Exploits | Yes |
| WebSocket Support | Yes |
| Force SSL | Yes |
| HTTP/2 | Yes |
| SSL Certificate | `pbs.internal (Custom)` |

Save changes and restart NPM if required.

### 12. Verify HTTPS Access
```
curl -Ik https://pbs.internal
```

Expected:

```
HTTP/2 200
server: openresty
content-type: text/html
```

Browser test: open `https://pbs.internal` and verify the certificate chain shows **cfolino Root CA**.

---

## Overview
- Uses **Ansible** to orchestrate certificate generation on the **CA host** and deployment to the **NPM Docker host**.
- Automates CSR creation, signing via internal CA, and secure certificate transfer.
- Supports **one-host-at-a-time** operation.
- Automatically restarts the **NPM Docker container**.
- Fully **non-interactive**, key-based authentication.

---

## Prerequisites
- The `ansible` user exists on both **CA** and **NPM** hosts.
- Passwordless SSH access is configured.
- The **CA server** (`ca.internal`) has the directory structure:

```
/home/ca/ca/sign-cert.sh
/home/ca/ca/pending-csrs/
/home/ca/ca/keys/
/home/ca/ca/signed-certs/
```

- The **NPM host** runs Docker and Nginx Proxy Manager.
- The `ansible` user belongs to both `docker` and `npm` groups.

---

## Validation
```bash
openssl x509 -in /home/npm/npm/data/custom_ssl/npm-<ID>/fullchain.pem -noout -subject -issuer -dates
```

Expected:

```
subject=O = cfolino, OU = homelab, CN = <FQDN>
issuer=C = US, O = Homelab, CN = cfolino Root CA
```

Browser validation: confirm HTTPS loads without redirect loops.

---

## Troubleshooting
| Issue | Cause | Resolution |
|--------|--------|-------------|
| ERR_TOO_MANY_REDIRECTS | Scheme mismatch in NPM | Set **Scheme: HTTPS** for PBS or add `X-Forwarded-Proto https;` |
| Missing certificate | Incorrect ID in `npm_cert_map` | Update mapping to match `/home/npm/npm/data/custom_ssl/npm-<ID>` |
| Permission denied | `ansible` missing group access | Add to `docker` and `npm` groups, reapply permissions |
| Docker restart fails | `ansible` not in docker group | `sudo usermod -aG docker ansible` |
| Bad interpreter (^M) | Windows line endings | `sed -i 's/\r$//' file && chmod +x file` |

---

## Workflow Summary
| Step | Description |
|------|--------------|
| Inventory & DNS | Define internal host and Pi-hole entries |
| Proxy Prep | Create dummy cert and map ID |
| Firewall | Allow NPM to backend service |
| Verification | Run Semaphore ping |
| CSR Generation | Performed on CA |
| Signing | Internal CA signs CSR |
| Deployment | Cert copied to NPM |
| NPM Reload | Docker container restarted |
| Apply & Verify | Attach cert and confirm HTTPS access |
