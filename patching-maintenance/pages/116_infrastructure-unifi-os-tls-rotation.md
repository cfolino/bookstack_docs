**System:** UniFi Dream Machine Pro (UDM-Pro)
**OS Version:** UniFi OS 3.x / 4.x
**Role:** `unifi_tls_rotate`
**Frequency:** Monthly (10th at 04:45)

---

### Overview
Automates the issuance and deployment of internal CA-signed certificates to the UniFi Dream Machine Pro. Unlike standard Linux servers, UniFi OS uses a UUID-based internal store for persistence. Direct modification of `/data/unifi-core/config/unifi-core.crt` is insufficient as it gets overwritten by the "authoritative" UUID file on reboot.

### Workflow Logic
1.  **Generation (On Device):** Generates a private key and CSR directly on the UDM to ensure the private key never leaves the appliance (except for backup).
2.  **Signing (Central CA):** Ansible fetches the CSR, signs it via the `cfolino Root CA` (intermediary script), and returns the signed CRT.
3.  **Persistence (UUID Store):** The signed cert is written to `/data/unifi-core/config/[UUID].crt`. This is the source of truth for UniFi OS.
4.  **Synchronization:** The playbook forcibly copies the UUID cert to `unifi-core.crt` and `unifi-core-direct.crt` to ensure immediate application without a full reboot.
5.  **Restart:** The `unifi-core` service is restarted via `systemd`.

### Configuration

#### Ansible Role Structure
Located at: `~/semaphore/roles/unifi_tls_rotate/`

* **`defaults/main.yml`**: Contains the UUID and Certificate Identity.
    ```yaml
    cert_name: <internal-host>
    unifi_tls_uuid: eaa7593b-a4c0-444e-bb7f-71a193c5cad6  # CAUTION: Changes on Factory Reset
    ```

* **`tasks/deploy.yml`**: Targets the UUID file.
* **`tasks/verify.yml`**: Loops on port 443 to verify the new cert is actively serving.

#### Network Requirements
* **SSH Access:** Root access to UDM (192.168.x.x) via SSH keys.
* **Inventory:** Host must be in `[unifi]` group with `ansible_become: false` (since user is already root).

### Troubleshooting & Validation

**1. Verify Filesystem Timestamps**
SSH into the UDM and confirm the active certificate and the UUID backup were updated at the same time:
```bash
ssh root@192.168.x.x
ls -la /data/unifi-core/config/
# Ensure unifi-core.crt and [UUID].crt have the same fresh timestamp.
```

**2. Verify Live HTTPS Socket**
Run this from any Linux host (e.g., Ansible controller) to confirm the web server has reloaded the new file:
```bash
openssl s_client -connect 192.168.x.x:443 < /dev/null 2>/dev/null | openssl x509 -noout -dates -subject
# Expected Output:
# notBefore=Feb  8 16:24:04 2026 GMT (Today's Date)
# subject=CN = <internal-host>
```

**3. Manual Rollback**
If the service fails to start (UI inaccessible), restore the backup files automatically created by the playbook:
```bash
ssh root@192.168.x.x
cp /data/unifi-core/config/unifi-core.crt.bak /data/unifi-core/config/unifi-core.crt
cp /data/unifi-core/config/unifi-core.key.bak /data/unifi-core/config/unifi-core.key
systemctl restart unifi-core
```
