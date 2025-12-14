## Purpose

Creates a **live full backup** of the OMV root partition using `fsarchiver`, with timestamped backup files and logging. It also **prunes old backups**, keeping only the most recent.

---

## Overview

* Backs up the root partition (e.g., `/dev/sdd2`) to a timestamped `.fsa` archive.
* Stores backups under a specified backup directory on a CIFS-mounted share.
* Logs progress and errors to `/var/log/omv_full_image_backup.log`.
* Keeps only the most recent full backup, deleting older ones.

---

##  Configuration Variables

| Variable          | Description                                    |
| ----------------- | ---------------------------------------------- |
| `BACKUP_BASE`     | Base directory for OMV backups                 |
| `FULL_BACKUP_DIR` | Directory for full image backups               |
| `LOGFILE`         | Path to log file                               |
| `ROOT_PARTITION`  | Partition device to backup (e.g., `/dev/sdd2`) |
| `BACKUP_FILE`     | Full backup filename with timestamp            |

---

##  Script Content

```bash
#!/bin/bash
set -euo pipefail

# === Configuration ===
BACKUP_BASE="/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups"
FULL_BACKUP_DIR="$BACKUP_BASE/full_image_backups"
LOGFILE="/var/log/omv_full_image_backup.log"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
ROOT_PARTITION="/dev/sdd2"
BACKUP_FILE="$FULL_BACKUP_DIR/omv_full_backup_$TIMESTAMP.fsa"

log() {
  echo "$(date +"%F %T") $*" | tee -a "$LOGFILE"
}

log "=================== Starting full OMV image backup ==================="

mkdir -p "$FULL_BACKUP_DIR"
log "Backup directory: $FULL_BACKUP_DIR"

log "Running live fsarchiver backup on $ROOT_PARTITION to $BACKUP_FILE"
fsarchiver savefs -v -j4 -A "$BACKUP_FILE" "$ROOT_PARTITION"

log "Backup completed: $BACKUP_FILE"

backup_files=( $(ls -1t "$FULL_BACKUP_DIR"/omv_full_backup_*.fsa 2>/dev/null) )
if [ ${#backup_files[@]} -gt 1 ]; then
  for oldfile in "${backup_files[@]:1}"; do
    rm -f "$oldfile"
    log "Deleted old full backup: $oldfile"
  done
fi

log "Pruning complete."
log "=================== Full OMV image backup finished ==================="
```

---

##  Usage Notes

* Run as root or via cron to perform scheduled full backups.
* Ensure the backup target (`BACKUP_BASE`) is mounted and has sufficient space.
* Adjust `ROOT_PARTITION` if your OMV root partition differs.
* Log file keeps a record of backup and pruning activity.

---

##  Workflow Summary

| Step                 | Description                                  |
| -------------------- | -------------------------------------------- |
| 1. Prepare directory | Creates backup directory if missing          |
| 2. Run fsarchiver    | Live backup of root partition to `.fsa`      |
| 3. Log completion    | Logs backup file name and timestamp          |
| 4. Prune old backups | Keeps only the newest backup, deletes others |

---
