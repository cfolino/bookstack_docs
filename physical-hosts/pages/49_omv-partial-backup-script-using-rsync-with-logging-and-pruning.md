## Purpose

Performs a selective backup of critical OpenMediaVault system directories and files using `rsync` with inclusion/exclusion rules.
Logs all operations, exports installed package list and OS version info, and prunes old backups and metadata files to keep storage usage manageable.

---

## Overview

* Uses `rsync` to back up selected directories and files to a timestamped backup directory.
* Copies critical config files separately to ensure completeness.
* Saves a list of manually installed packages and OS version information.
* Deletes older backups and metadata files, keeping only the most recent.

---

## Configuration Variables

| Variable      | Description                              |
| ------------- | ---------------------------------------- |
| `BACKUP_BASE` | Base directory where backups are stored  |
| `LOGFILE`     | Log file path for backup operations      |
| `RSYNC_DEST`  | Target backup directory with timestamp   |
| `PKG_LIST`    | File path to save installed package list |
| `OS_VERSION`  | File path to save OS version info        |

---

## Rsync Include Rules

Backs up the following directories (with all contents):

* `/boot/` and `/boot/efi/`
* `/etc/` and subdirectories including `/etc/openmediavault/`
* `/home/`
* `/root/`
* `/usr/local/`
* `/var/lib/openmediavault/`
* `/var/spool/cron/`
* `/var/log/`
* `/lib/udev/rules.d/`
* `/etc/systemd/`

All other files and directories are excluded from this backup.

---

## Script Content

```bash
#!/bin/bash
set -euo pipefail

# === Configuration ===
BACKUP_BASE="/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups"
LOGFILE="/var/log/omv_backup.log"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
RSYNC_DEST="$BACKUP_BASE/os_backup_$TIMESTAMP"
PKG_LIST="$BACKUP_BASE/manual_packages_$TIMESTAMP.txt"
OS_VERSION="$BACKUP_BASE/os_version_$TIMESTAMP.txt"

# Define includes for rsync
INCLUDES=(
  "--include=/boot/"
  "--include=/boot/efi/"
  "--include=/etc/"
  "--include=/etc/***"
  "--include=/etc/openmediavault/"
  "--include=/etc/openmediavault/***"
  "--include=/home/"
  "--include=/home/***"
  "--include=/root/"
  "--include=/root/***"
  "--include=/usr/local/"
  "--include=/usr/local/***"
  "--include=/var/lib/openmediavault/"
  "--include=/var/lib/openmediavault/***"
  "--include=/var/spool/cron/"
  "--include=/var/spool/cron/***"
  "--include=/var/log/"
  "--include=/var/log/***"
  "--include=/lib/udev/rules.d/"
  "--include=/lib/udev/rules.d/***"
  "--include=/etc/systemd/"
  "--include=/etc/systemd/***"
  "--exclude=*"
)

# Logging function
log() {
  echo "$(date +"%F %T") $*" | tee -a "$LOGFILE"
}

log "=================== Starting OMV backup ==================="

# Create backup directory
mkdir -p "$RSYNC_DEST"
log "Created backup directory: $RSYNC_DEST"

# Rsync backup with includes and deletes
log "Starting rsync backup..."
rsync -aAXHv --delete "${INCLUDES[@]}" / "$RSYNC_DEST"
if [ $? -ne 0 ]; then
  log "ERROR: rsync backup failed."
  exit 1
fi
log "rsync backup completed successfully."

# Copy critical system files separately with permissions preserved
critical_files=(
  /etc/fstab
  /etc/openmediavault/config.xml
  /etc/hostname
  /etc/hosts
  /etc/passwd
  /etc/group
  /etc/shadow
  /etc/gshadow
)

log "Copying critical system files..."
for file in "${critical_files[@]}"; do
  if [ -f "$file" ]; then
    cp --preserve=mode,ownership,timestamps "$file" "$RSYNC_DEST/"
    log "Copied $file"
  else
    log "WARNING: $file not found, skipping"
  fi
done

# Export manually installed package list
log "Exporting manually installed package list..."
apt-mark showmanual > "$PKG_LIST"
log "Package list saved to $PKG_LIST"

# Export OS version info
log "Exporting OS version info..."
if lsb_release -a &>/dev/null; then
  lsb_release -a > "$OS_VERSION" 2>/dev/null
else
  cat /etc/os-release > "$OS_VERSION"
fi
log "OS version info saved to $OS_VERSION"

# Prune old backups and files: keep only the most recent backup directory and related package/version files
log "Pruning old backups..."

# Prune os_backup directories - keep newest only
backup_dirs=( $(ls -1dt "$BACKUP_BASE"/os_backup_*/ 2>/dev/null) )
if [ ${#backup_dirs[@]} -gt 1 ]; then
  for olddir in "${backup_dirs[@]:1}"; do
    rm -rf "$olddir"
    log "Deleted old backup directory: $olddir"
  done
fi

# Prune manual_packages files - keep newest only
pkg_files=( $(ls -1t "$BACKUP_BASE"/manual_packages_*.txt 2>/dev/null) )
if [ ${#pkg_files[@]} -gt 1 ]; then
  for oldfile in "${pkg_files[@]:1}"; do
    rm -f "$oldfile"
    log "Deleted old package list file: $oldfile"
  done
fi

# Prune os_version files - keep newest only
osver_files=( $(ls -1t "$BACKUP_BASE"/os_version_*.txt 2>/dev/null) )
if [ ${#osver_files[@]} -gt 1 ]; then
  for oldfile in "${osver_files[@]:1}"; do
    rm -f "$oldfile"
    log "Deleted old OS version file: $oldfile"
  done
fi

log "Pruning complete."
log "=================== OMV backup finished ==================="
```

---

## Usage Notes

* Run as root or via cron to regularly back up critical OMV configuration and user data.
* Adjust `BACKUP_BASE` to the correct mounted backup location.
* Monitor the log file `/var/log/omv_backup.log` for success or errors.
* Make sure `apt-mark`, `lsb_release`, and `rsync` are installed and accessible.

---

## Workflow Summary

| Step                          | Description                                  |
| ----------------------------- | -------------------------------------------- |
| 1. Prepare backup directory   | Create timestamped backup directory          |
| 2. Rsync selective backup     | Copy configured directories and files        |
| 3. Copy critical files        | Individually copy system config files        |
| 4. Export package and OS info | Save manually installed packages and OS info |
| 5. Prune older backups        | Delete older backup directories and metadata |

---
