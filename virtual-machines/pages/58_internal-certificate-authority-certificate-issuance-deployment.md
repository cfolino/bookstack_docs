---

This page documents the automated workflow used to generate, sign, and deploy internal TLS certificates from the homelab’s private Certificate Authority (CA) to Nginx Proxy Manager (NPM) using Ansible.

The automation ensures that every internal service exposed through NPM receives a valid, trusted certificate issued by the internal CA, with minimal manual handling and clear verification points.

---

## Purpose

This automation exists to:

- Provide trusted HTTPS for all internal services
- Eliminate manual certificate generation and copying
- Centralize trust in an internal CA
- Ensure consistent certificate deployment in NPM
- Support certificate rotation without service redesign

Certificates are issued **one service at a time** by design to avoid ambiguity and reduce blast radius.

---

## Architecture Overview

Certificate issuance is orchestrated by the Ansible control node and involves three systems:

### Certificate Authority (CA)
- Host: `<internal-host>`
- Generates private keys and CSRs
- Signs certificates using the internal root CA
- Archives issued certificates
- Performs signing without sudo

### Ansible Control Node
- Coordinates the workflow
- Validates prerequisites
- Fetches signed artifacts temporarily
- Builds fullchain bundles
- Cleans up all staging files

### Nginx Proxy Manager (NPM)
- Stores certificates in per-ID directories
- Terminates TLS for internal services
- Is restarted automatically to activate new certificates

---

## Entry Playbook

```bash
ansible-playbook -i inventory/hosts.yml playbooks/issue_and_deploy_cert.yml
```

The playbook prompts for:
- **FQDN** (certificate common name)
- **Service IP** (added as SAN)

Example:
```text
<internal-host>
192.168.x.x
```

---

## Prerequisites

### Inventory & Access
- Target host exists in Ansible inventory using its **internal hostname**
- Passwordless SSH access is configured
- The `ansible` user exists on:
  - CA host
  - NPM host

### Internal DNS
Two DNS records must exist in Pi-hole:

| Name | Purpose |
|-----|--------|
| `<service>.cfolino.com` | Public-facing name → NPM |
| `<service><internal-host>` | Direct backend access |

Verify:
```bash
dig +short <internal-host>
dig +short <internal-host>
```

---

## NPM Certificate Mapping

Each certificate is mapped to an NPM numeric ID.

Defined in:
```text
inventory/host_vars/npm.yml
```

Example:
```yaml
npm_cert_map:
  <internal-host>: 3
  <internal-host>: 6
  <internal-host>: 14
```

This mapping ensures certificates are deployed into the correct NPM directory:

```text
/home/npm/npm/data/custom_ssl/npm-<ID>/
```

---

## Workflow (End-to-End)

### 1. Preflight Validation
The playbook verifies:
- The FQDN exists in `npm_cert_map`
- NPM is reachable
- CA is reachable

Execution aborts early if any requirement is missing.

---

### 2. Key & CSR Generation (CA Host)

- A 4096-bit RSA private key is generated
- A CSR is created using:
  - CN = FQDN
  - O / OU = homelab identity
- CSR creation is **idempotent**

Keys and CSRs never leave the CA during generation.

---

### 3. Certificate Signing

The CA signs the CSR using a custom signing script:

```text
/home/ca/ca/sign-cert.sh
```

Features:
- Adds DNS and IP SANs
- Archives issued certificates
- Produces deterministic output
- Emits stdout/stderr for auditing

---

### 4. Artifact Retrieval & Fullchain Assembly

The control node:
- Fetches the signed certificate
- Fetches the private key (if present)
- Builds a `fullchain.pem`
- Stages files temporarily in:
  ```text
  /tmp/ansible-cert-<fqdn>
  ```

---

### 5. Deployment to NPM

Files are deployed to:
```text
/home/npm/npm/data/custom_ssl/npm-<ID>/
├── fullchain.pem
└── privkey.pem
```

Permissions:
- `fullchain.pem`: `0644`
- `privkey.pem`: `0600`

---

### 6. NPM Reload

The NPM Docker container is restarted automatically to activate the new certificate.

---

### 7. Cleanup

All temporary files on the control node are removed.
No cryptographic material remains outside intended locations.

---

## Applying the Certificate in NPM

After deployment:
1. Edit the corresponding proxy host
2. Select the new **Custom Certificate**
3. Enable:
   - HTTPS
   - Force SSL
   - HTTP/2
4. Save changes

---

## Verification

### Certificate Inspection
```bash
openssl x509 \
  -in /home/npm/npm/data/custom_ssl/npm-<ID>/fullchain.pem \
  -noout -subject -issuer -dates
```

Expected:
```text
subject=O=cfolino, OU=homelab, CN=<FQDN>
issuer=C=US, O=Homelab, CN=cfolino Root CA
```

### Service Test
```bash
curl -Ik https://<service>.cfolino.com
```

Browser verification should show **cfolino Root CA** as issuer.

---

## Security Guarantees

- Private keys are generated on the CA
- Signing occurs without root
- Keys are never persisted on the control node
- Certificates are deployed only to NPM
- Trust is anchored exclusively to the internal CA

---

## Common Issues & Resolution

| Issue | Cause | Resolution |
|-----|------|-----------|
| Missing cert | Wrong NPM ID | Update `npm_cert_map` |
| Redirect loop | Scheme mismatch | Set proxy scheme to HTTPS |
| Permission denied | Missing group access | Ensure `ansible` is in `docker` and `npm` groups |
| Script errors | Windows line endings | `sed -i 's/\r$//' sign-cert.sh` |

---

## Design Principles

- Explicit trust boundaries
- Deterministic automation
- No hidden state
- Minimal manual intervention
- Clear failure modes

---

## Related Pages

- Ansible Inventory & Group Design
- Notification & Email Automation
- Nginx Proxy Manager
- Internal DNS Architecture

---
