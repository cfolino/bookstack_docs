## Overview

This document describes the automated TLS certificate lifecycle for internal services exposed through **Nginx Proxy Manager (NPM)** using the **cfolino.com internal Certificate Authority**.

Certificates are:

- Issued and signed via Ansible
- Short-lived (35 days)
- Renewed automatically based on remaining lifetime
- Executed via Semaphore on a dedicated TLS maintenance day
- Isolated per certificate (one cert per job)

This design prioritizes **safety, auditability, and zero surprise behavior**.

---

## Design Principles

- **One certificate per run**
- **One Semaphore job per certificate**
- **No loops or bulk operations**
- **Explicit renewal gates**
- **Fail-fast input validation**
- **No automatic force overrides**

All behavior is deterministic and observable.

---

## Playbook

**Playbook path:**
```
playbooks/issue_and_deploy_cert.yml
```

**Target host:**
```
npm
```

---

## Variables and Inputs

### External (runtime) variables

| Variable | Required | Purpose |
|--------|--------|--------|
| `cert_fqdn` | Yes | Fully qualified domain name of the certificate to issue |
| `force_renewal` | No | One-time override to force renewal (migration only) |

### Internal variables

| Variable | Description |
|--------|-------------|
| `cert_name` | Effective certificate name derived from `cert_fqdn` |
| `cert_ip` | IP address to include as SAN (from `npm_certificates`) |
| `cert_renewal_required` | Computed renewal decision |

---

## Renewal Logic

A certificate is renewed when **any** of the following conditions are true:

- The certificate does not exist
- The certificate expires in less than **10 days**
- A one-time `force_renewal=true` override is explicitly provided

Default behavior **never forces renewal**.

---

## Input Validation

The playbook enforces strict preflight validation on `cert_name`:

- Must be a string
- Must not contain spaces
- Must not contain `/`
- Must not contain `..`
- Must start and end with an alphanumeric character
- Length must be ≤ 253 characters

Invalid input fails immediately before any CA or NPM operations.

---

## Data Sources

Certificate metadata is stored in inventory data:

### `npm_cert_map`
Maps FQDNs to NPM certificate IDs.

### `npm_certificates`
Defines certificate-specific attributes such as IP SANs.

Example:

```yaml
npm_certificates:
  <internal-host>:
    ip: 192.168.x.x

  <internal-host>:
    ip: 192.168.x.x
```

---

## Certificate Issuance Flow

1. Validate input and mappings
2. Check existing certificate expiry on the CA
3. Decide renewal based on lifetime or override
4. Generate CSR and private key (idempotent)
5. Sign certificate via internal CA
6. Fetch signed cert to controller
7. Build fullchain
8. Deploy cert to NPM custom SSL directory
9. Restart NPM container
10. Cleanup temporary files

---

## One-Time Migration: Long-Lived to Short-Lived Certs

When migrating an existing long-lived certificate into the 35-day rotation, a **single forced run** is performed.

### Example (BookStack migration)

```bash
ansible-playbook playbooks/issue_and_deploy_cert.yml -i inventory \
  -e cert_fqdn=<internal-host> \
  -e force_renewal=true
```

After this run:
- The certificate lifetime is reset to 35 days
- The service joins the normal monthly renewal cycle
- `force_renewal` must **not** be used again

---

## Verification

### Check certificate served by NPM

```bash
openssl s_client -connect <internal-host>:443 -servername <internal-host> </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer
```

---

## Semaphore Scheduling Model

- Each certificate has its **own Semaphore job**
- All TLS jobs run on the dedicated TLS maintenance day
- Jobs are isolated and non-overlapping
- No job chaining

---

## Active TLS Jobs

| Service | FQDN | Semaphore Job |
|------|------|---------------|
| Grafana | <internal-host> | `tls-grafana-cert` |
| BookStack | <internal-host> | `tls-bookstack-cert` |

---

## Operational Notes

- No certificates are renewed outside the renewal window
- No certificates are rotated silently
- All changes are logged and observable
- TLS automation is independent from patching and infrastructure maintenance

---

## Status

✅ TLS automation fully deployed
✅ Short-lived cert lifecycle enforced
✅ Browser trust verified
✅ Semaphore scheduling complete
