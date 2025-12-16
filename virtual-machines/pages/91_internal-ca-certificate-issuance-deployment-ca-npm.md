---

## Purpose

This page documents the **end-to-end certificate lifecycle** used to:

- Generate a TLS private key and CSR
- Sign the certificate using the internal Certificate Authority (CA)
- Deploy the resulting certificate to **Nginx Proxy Manager (NPM)**
- Reload NPM so the new certificate becomes active

This workflow is designed to be **secure, deterministic, and auditable**, with explicit control over where cryptographic material is generated, signed, transported, and stored.

---

## Scope

### Applies to

- Internally trusted TLS certificates
- Services fronted by **Nginx Proxy Manager**
- Certificates issued by the internal CA (`<internal-host>`)
- Manual, operator-initiated certificate issuance

### Explicit exclusions

This workflow does **not**:

- Use Let’s Encrypt or any public CA
- Automatically rotate certificates
- Issue wildcard certificates
- Deploy certificates directly to application hosts
- Perform unattended issuance

---

## Entry Playbook

```text
playbooks/issue_and_deploy_cert.yml
```

---

## High-Level Architecture

```text
Ansible Controller
        |
        | (SSH / Ansible)
        v
Internal CA  ──► signs certificate
        |
        | (fetch)
        v
Ansible Controller (temporary staging)
        |
        | (copy)
        v
Nginx Proxy Manager (custom_ssl directory)
```

---

## Inventory & Variable Model

### Certificate Mapping (NPM)

Each FQDN must be mapped to an NPM certificate ID:

```yaml
# inventory/host_vars/npm.yml
npm_data_root: /home/npm/npm/data/custom_ssl

npm_cert_map:
  <internal-host>: 1
  <internal-host>: 2
```

This mapping determines **where the certificate is deployed** inside NPM.

---

### CA Variables (Defined on NPM host vars)

The CA paths and signing script are referenced explicitly:

```yaml
ca_csr_dir: /home/ca/ca/pending-csrs
ca_signed_dir: /home/ca/ca/signed-certs
ca_sign_script: /home/ca/ca/sign-cert.sh
```

The CA archive root is normalized in the playbook:

```text
/home/ca/ca/archive
```

---

## Operator Input (Interactive)

This playbook is intentionally interactive.

At runtime, the operator is prompted for:

- **Certificate FQDN**
  ```text
  <internal-host>
  ```
- **Service IP address** (used for SAN)
  ```text
  192.168.x.x
  ```

These values are used to build the CSR and SAN entries.

---

## Execution Flow (Authoritative)

### 1. Preflight Validation

Before any cryptographic operations occur:

- Validate that `npm_cert_map` exists
- Validate that the requested FQDN has an NPM mapping
- Confirm reachability of:
  - NPM host
  - CA host

Failures here abort execution immediately.

---

### 2. Controller Staging Directory

A temporary directory is created on the Ansible controller:

```text
/tmp/ansible-cert-<fqdn>
```

Properties:
- Mode `0700`
- Used only for transient transport
- Automatically removed at the end of the run

---

### 3. CA Directory Preparation

On the CA host:

- Required directories are ensured to exist
- Directories are group-writable to allow signing without sudo

Directories include:
- Private keys
- CSRs
- Signed certificates
- Archive storage

---

### 4. Private Key & CSR Generation (on CA)

**Key point:**
Private keys are generated **on the CA host**, not on the controller or NPM.

Command behavior:
- RSA 4096-bit key
- No passphrase (`-nodes`)
- Idempotent (skipped if CSR already exists)

Artifacts created:
- `{{ cert_name }}.key`
- `{{ cert_name }}.csr`

---

### 5. Certificate Signing (on CA)

The CSR is signed using the CA’s signing script:

```text
sign-cert.sh <csr> <crt> <fqdn> <ip>
```

Features:
- SAN includes DNS + IP
- Certificate is archived automatically
- Signing is performed **without sudo**
- Stdout and stderr are captured and logged

---

### 6. Artifact Retrieval (Controller)

The controller fetches:

- The signed certificate (`.crt`)
- The private key (`.key`) if present

These files exist **only temporarily** on the controller.

---

### 7. Fullchain Construction (Controller)

The controller builds:

```text
<fqdn>.fullchain.pem
```

Contents:
- Leaf certificate
- Optional CA chain (if configured)

This file is what NPM consumes.

---

### 8. Deployment to NPM

On the NPM host:

- Certificate directory is ensured:
  ```text
  {{ npm_data_root }}/npm-<cert_id>
  ```
- Files deployed:
  - `fullchain.pem` (0644)
  - `privkey.pem` (0600, only if staged)

No certificate logic exists inside NPM itself — it simply reads files.

---

### 9. NPM Reload

The NPM Docker container is restarted:

```yaml
community.docker.docker_container:
  restart: true
```

This forces NPM to reload the new certificate.

---

### 10. Confirmation & Cleanup

- A confirmation message is printed with:
  - FQDN
  - Certificate ID
  - Deployment path
- Temporary controller files are removed

No private key material remains outside CA + NPM.

---

## Security Properties

This workflow guarantees:

- Private keys are never generated on NPM
- CA signing is centralized and auditable
- Controller staging is ephemeral
- Permissions are explicitly set
- Certificates are mapped deterministically to NPM IDs
- No implicit trust or wildcard behavior

---

## Execution Command

```bash
ansible-playbook playbooks/issue_and_deploy_cert.yml
```

Dry-run mode is intentionally **not supported**, because certificate issuance is a stateful operation.

---

## Post-Deployment Verification

From a client machine:

```bash
openssl s_client -connect <fqdn>:443 -servername <fqdn>
```

Confirm:
- Correct certificate CN
- Correct SAN entries
- Internal CA trust chain

---

## Failure Modes & Recovery

| Scenario | Recovery |
|--------|----------|
| Missing npm_cert_map entry | Add mapping, re-run |
| CA unreachable | Fix connectivity, re-run |
| NPM container fails to restart | Restart manually, inspect logs |
| Wrong cert deployed | Fix mapping, re-issue |

Because certificates are archived on the CA, rollback is always possible.

---

## Explicit Non-Goals

This workflow does **not**:

- Automate renewals
- Replace application-level TLS
- Expose private keys to end users
- Integrate with ACME or public CAs

---

## Summary

This certificate automation pipeline provides **secure, repeatable internal TLS** using a dedicated CA and deterministic deployment into Nginx Proxy Manager.

All cryptographic operations are explicit, auditable, and controlled, making this workflow suitable for long-term homelab operations and professional demonstration.

---
