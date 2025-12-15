## Purpose

Creates a recursive ZFS snapshot of the specified pool, sends it compressed over SSH to a remote OpenMediaVault (OMV) NAS, and cleans up local snapshots.
Implements retention by deleting older backups on the remote NAS.

---

## Overview

* Uses `zfs snapshot -r` to create a recursive snapshot with a timestamped name.
* Streams the snapshot via `zfs send` piped to `zstd` compression, then transfers it securely to OMV using SSH.
* Deletes the local snapshot after successful transfer.
* On OMV, retains only the newest backup by deleting older files matching the backup pattern.

---

## Configuration Variables

| Variable    | Description                                     |
| ----------- | ----------------------------------------------- |
| `POOL`      | ZFS pool name to snapshot (e.g., `rpool`)       |
| `TIMESTAMP` | Timestamp string used in snapshot and filenames |
| `SNAP`      | Snapshot name including timestamp               |
| `DEST_NAME` | Filename of the compressed snapshot backup      |
| `OMV_USER`  | SSH user for OMV backup server                  |
| `OMV_HOST`  | OMV backup server IP address                    |
| `OMV_PATH`  | Destination path on OMV for backup files        |
| `SSH_KEY`   | SSH private key path used for authentication    |
| `LOG_FILE`  | Local log file path for backup operation        |

---

## Script Content

```bash
#!/bin/bash
set -euo pipefail

# === Configuration ===
POOL="rpool"
TIMESTAMP="$(date +%F_%H-%M-%S)"
SNAP="full-${TIMESTAMP}"
DEST_NAME="rpool_full_${TIMESTAMP}.zfs.zst"
OMV_USER="pvesvc"
OMV_HOST="192.168.15.225"
OMV_PATH="/srv/dev-disk-by-uuid-<uuid-redacted>/backup_cifs/pve_backups"
SSH_KEY="/root/.ssh/pve_to_omv_backup_ed25519"
LOG_FILE="/var/log/full_rpool_backup.log"

echo "[$(date)] Starting ZFS full backup..." | tee -a "$LOG_FILE"

# === Create recursive snapshot ===
echo "[$(date)] Creating snapshot $POOL@$SNAP..." | tee -a "$LOG_FILE"
zfs snapshot -r "${POOL}@${SNAP}"

# === Send and compress snapshot ===
echo "[$(date)] Sending snapshot to OMV as ${DEST_NAME}..." | tee -a "$LOG_FILE"
zfs send -R "${POOL}@${SNAP}" | zstd -T0 -19 | ssh -i "$SSH_KEY" "${OMV_USER}@${OMV_HOST}" \
    "cat > '${OMV_PATH}/${DEST_NAME}'"

# === Clean up local snapshot ===
echo "[$(date)] Destroying local snapshot $POOL@$SNAP..." | tee -a "$LOG_FILE"
zfs destroy -r "${POOL}@${SNAP}"

# === Retention: remove all older backups ===
echo "[$(date)] Removing older backups on OMV..." | tee -a "$LOG_FILE"
ssh -i "$SSH_KEY" "${OMV_USER}@${OMV_HOST}" <<EOF
cd "${OMV_PATH}" && \
ls -1 rpool_full_*.zfs.zst | grep -v "${DEST_NAME}" | xargs -r rm -f
EOF

echo "[$(date)] Full ZFS backup completed successfully." | tee -a "$LOG_FILE"
```

---

## Usage Notes

* Run as root or a privileged user with ZFS and SSH access configured.
* Ensure SSH key `$SSH_KEY` has proper permissions and access to the OMV backup host.
* Requires `zstd` installed locally for compression.
* Backup retention is enforced on the remote OMV server by deleting older compressed snapshot files.
* Monitor `/var/log/full_rpool_backup.log` for backup status and errors.

---

## Workflow Summary

| Step                          | Description                                                |
| ----------------------------- | ---------------------------------------------------------- |
| 1. Create recursive snapshot  | Creates timestamped recursive ZFS snapshot                 |
| 2. Send and compress snapshot | Streams snapshot compressed to OMV via SSH                 |
| 3. Destroy local snapshot     | Removes snapshot locally after successful send             |
| 4. Remote retention           | Deletes older backup files on OMV, keeping only the newest |

---
