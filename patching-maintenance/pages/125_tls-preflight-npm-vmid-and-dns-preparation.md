# TLS Automation Preflight — Required Setup Before Issuance

## Purpose

This document defines the **mandatory preflight steps** that must be completed before a service is eligible for automated TLS issuance and deployment.

TLS automation **does not create infrastructure**.
It assumes all identities already exist and is intentionally designed to **fail fast** if any prerequisite is missing.

This preflight establishes **authoritative identity mappings** for:

- Proxmox VM (`vmid`)
- Inventory host identity
- Nginx Proxy Manager (NPM) proxy
- NPM certificate object (numeric ID)
- Certificate SAN IP
- Internal DNS resolution

If any step in this document is skipped or incomplete, **TLS automation must not be run**.

---

## When This Procedure Is Used

This procedure is performed:

- Once per service / FQDN
- After the VM is created and reachable
- Before creating a Semaphore TLS job
- Before running `issue_and_deploy_cert.yml`

This procedure is **not** part of certificate renewal.

---

## Required Information (Before You Begin)

You must know **all** of the following before starting:

- Inventory hostname (VM name)
- Proxmox VMID
- Service FQDN (example: `<internal-host>`)
- Backend service IP address
- Backend service port
- NPM proxy IP address
- Confirmation the VM is reachable and the service is running

If any item is unknown, **stop here**.

---

## Step 1 — Create Host Inventory File (VMID Required)

Every VM **must** have a dedicated host inventory file before TLS automation.

### Location

```
/home/ansible/ansible/inventory/host_vars/<hostname>.yml
```

### Required Contents

```yaml
vmid: <PROXMOX_VMID>
```

Example:

```yaml
vmid: 105
```

### Why This Matters

The `vmid` is a **hard requirement** and is used by:

- Proxmox lifecycle operations
- Patch and reboot automation
- Safety and validation checks

If this file or variable does not exist, **automation must not proceed**.

---

## Step 2 — Create the NPM Proxy Host

In the **Nginx Proxy Manager UI**:

1. Create a new **Proxy Host**
2. **Domain Names**
   ```
   <service>.cfolino.com
   ```
3. **Forward Hostname / IP**
   ```
   <service IP>
   ```
4. **Forward Port**
   ```
   <service port>
   ```
5. **Scheme**
   - HTTP (typical for internal services)
6. Enable required options as appropriate:
   - Websockets
   - Block Common Exploits
7. Save

At this point:

- The proxy identity exists
- TLS is not yet automated

---

## Step 3 — Upload a Placeholder Certificate

TLS automation requires an **existing certificate object** in NPM.

In **NPM → SSL Certificates**:

1. Add **Custom Certificate**
2. **Name**
   ```
   <service>-placeholder
   ```
   Example:
   ```
   bookstack-placeholder
   ```
3. Upload any valid certificate and key
   (self-signed is acceptable)
4. Save

This creates the **certificate identity** inside NPM.

---

## Step 4 — Attach the Placeholder Certificate to the Proxy

1. Edit the proxy host
2. Go to **SSL**
3. Select certificate:
   ```
   <service>-placeholder
   ```
4. Save

At this point:

- Proxy exists
- Certificate object exists
- Proxy ↔ certificate binding exists

---

## Step 5 — Discover the NPM Certificate ID (Authoritative)

TLS automation relies on the **internal numeric certificate ID** assigned by NPM.

To obtain it:

1. In **NPM → SSL Certificates**
2. Click the placeholder certificate
3. Observe the browser URL

Example:

```
https://internal.example/nginx/certificates/6
```

👉 **`6` is the certificate ID**

This ID:

- Is stable
- Must be recorded exactly
- Must not be guessed

---

## Step 6 — Record the Certificate ID in Inventory

Update the NPM inventory mapping.

### Location

```
/home/ansible/ansible/inventory/group_vars/npm.yml
```

### Required Entry

```yaml
npm_cert_map:
  <internal-host>: 6
```

Only record IDs observed directly from the NPM UI.

---

## Step 7 — Define the Certificate SAN IP

The SAN IP is injected during CSR signing and **must** be defined.

In the same inventory file:

```yaml
npm_certificates:
  <internal-host>:
    ip: 192.168.x.x
```

This IP is typically:

- The backend service IP
- Or another intentional SAN value

---

## Step 8 — Update Internal DNS

Update **internal DNS** (Pi-hole / Unbound):

- Record Type: `A`
- Hostname:
  ```
  <service>.cfolino.com
  ```
- IP Address:
  ```
  NPM proxy IP
  ```

### Verify DNS Resolution

From the Ansible controller:

```bash
dig <service>.cfolino.com
```

Expected result:

- Resolves to the NPM IP
- No external lookup required

---

## Step 9 — Verify HTTP Reachability

Before issuing TLS, confirm the proxy functions without SSL.

```bash
curl -I http://<service>.cfolino.com
```

Expected:

- HTTP response from the backend service
- No routing or connection errors

---

## Step 10 — Eligibility Checklist (Do Not Skip)

A service is eligible for TLS automation **only if all of the following are true**:

- `host_vars/<hostname>.yml` exists
- `vmid` is defined
- NPM proxy host exists
- Placeholder certificate exists
- Placeholder certificate is attached to the proxy
- Certificate ID is recorded in `group_vars/npm.yml`
- SAN IP mapping exists
- Internal DNS resolves correctly
- HTTP reachability is confirmed

If **any** item is missing, **do not run TLS automation**.

---

## Next Step

Once all preflight steps are complete, proceed to:

**Issue and Deploy TLS Certificate**

That process assumes all identity and inventory requirements are satisfied and will intentionally fail fast otherwise.
