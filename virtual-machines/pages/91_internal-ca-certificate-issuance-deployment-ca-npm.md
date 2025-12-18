## Purpose

This page documents the **end-to-end certificate lifecycle** used to:

- Generate a TLS private key and CSR
- Sign the certificate using the internal Certificate Authority (CA)
- Deploy the resulting certificate to **Nginx Proxy Manager (NPM)**
- Reload NPM so the new certificate becomes active

This workflow is designed to be **secure, deterministic, and auditable**, with explicit control over where cryptographic material is generated, stored, and deployed.

---

## Scope

### Applies to

- Internally trusted TLS certificates
- Services fronted by **Nginx Proxy Manager**
- Certificates issued by the internal CA (`<internal-host>`)

### Explicit exclusions

This workflow does **not**:

- Use ACME or public certificate authorities
- Perform wildcard certificate issuance
- Automatically rotate certificates on a schedule
- Manage trust distribution to client devices

---

## Entry Playbook

```text
playbooks/issue_and_deploy_cert.yml
```

---

## High-Level Workflow

1. Validate inventory mappings and connectivity
2. Generate a private key and CSR on the CA host
3. Sign the CSR using the internal CA
4. Fetch the signed certificate to the Ansible controller
5. Assemble a full certificate chain
6. Deploy the certificate and key to the NPM host
7. Restart the NPM container to activate the certificate
8. Clean up temporary artifacts

---

## Inventory and Mapping Requirements

### NPM Certificate Mapping

Each proxied service must have a **static certificate ID mapping** defined on the NPM host:

```yaml
npm_cert_map:
  <internal-host>: 3
  <internal-host>: 7
  <internal-host>: 4
  <internal-host>: 6
  <internal-host>: 14
```

This ID must match the directory:

```text
/home/npm/npm/data/custom_ssl/npm-<ID>
```

---

## Playbook Definition (Authoritative)

```yaml
- name: Generate, sign, and deploy certificate from CA to NPM
  hosts: npm
  gather_facts: false

  vars_prompt:
    - name: cert_name
      prompt: "Enter the FQDN (e.g. <internal-host>)"
      private: no
    - name: cert_ip
      prompt: "Enter the host IP (e.g. 192.168.x.x)"
      private: no
```

The playbook intentionally operates **one certificate at a time** to reduce blast radius and ensure traceability.

---

## Certificate Generation (CA Host)

- Private keys and CSRs are generated **only on the CA host**
- Keys are never generated on the NPM host
- CSR creation is idempotent

Key locations:

```text
/home/ca/ca/keys/
/home/ca/ca/pending-csrs/
/home/ca/ca/signed-certs/
/home/ca/ca/archive/
```

---

## Signing Process

- Signing is performed using `sign-cert.sh`
- SANs include:
  - FQDN
  - Explicit IP address
- All issued certificates are archived for audit purposes

The CA signing step does **not** require sudo.

---

## Deployment to NPM

Deployment actions:

- Fullchain assembled on the controller
- Files copied into the mapped NPM cert directory
- Permissions applied correctly
- NPM container restarted to load the new certificate

```text
fullchain.pem
privkey.pem
```

---

## Validation

### On the NPM Host

```bash
openssl x509 \
  -in /home/npm/npm/data/custom_ssl/npm-<ID>/fullchain.pem \
  -noout -subject -issuer -dates
```

Expected issuer:

```text
CN = cfolino Root CA
```

---

## Failure Modes and Recovery

| Issue | Likely Cause | Resolution |
|------|-------------|------------|
| Certificate not applied | Incorrect cert ID mapping | Verify `npm_cert_map` |
| Redirect loop | HTTP/HTTPS mismatch | Force HTTPS in NPM |
| Permission denied | Incorrect ownership | Fix file permissions |
| Container restart fails | Missing docker group | Add user to docker group |

---

## Security Properties

This workflow guarantees:

- Private keys never leave the CA host unencrypted
- Explicit inventory-controlled deployment
- Deterministic, repeatable certificate issuance
- Full audit trail of signed certificates

---

## Summary

Internal certificate issuance is treated as **infrastructure**, not convenience automation.

Each certificate is generated, signed, deployed, and activated through a controlled, auditable process that integrates tightly with Ansible, the internal CA, and Nginx Proxy Manager.
