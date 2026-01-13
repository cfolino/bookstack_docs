## Purpose

Creates a compressed tarball backup of essential Proxmox VE OS directories (excluding VM disk images and containers), transfers it securely to an OMV NAS via SSH, and manages retention by keeping only the latest 3 backups on the OMV server.

---

## Overview

* Archives key Proxmox system directories to a timestamped tar.gz file in `/tmp`.
* Uses `rsync` over SSH with a specified private key for secure transfer to OMV backup share.
* Cleans up local temporary backup file after transfer.
* On OMV, limits stored backups to the 3 most recent, deleting older ones to conserve space.
* Logs all steps and errors to a local logfile `/var/log/pve_backup.log`.

---

## Configuration Variables

| Variable        | Description                                                                |
| --------------- | -------------------------------------------------------------------------- |
| `BACKUP_NAME`   | Timestamped backup tarball filename                                        |
| `BACKUP_DIR`    | Remote OMV directory to store backups                                      |
| `LOG_FILE`      | Local log file path                                                        |
| `EXCLUDES`      | Directories excluded from backup tarball                                   |
| SSH private key | Hardcoded to `/root/.ssh/pve_to_omv_backup_ed25519` for secure connections |
| OMV SSH user    | `pvesvc` configured user on OMV server                                     |
| OMV IP address  | `192.168.x.x` (example)                                                 |

---

## Script Content

```bash
#!/bin/bash

# Variables
BACKUP_NAME="pve_backup_$(date +%Y-%m-%d_%H-%M-%S).tar.gz"
BACKUP_DIR="/srv/dev-disk-by-uuid-<uuid-redacted>/backup_cifs/pve_backups"
LOG_FILE="/var/log/pve_backup.log"

# Directories to exclude (excluding VM disk images, LXC containers, and QEMU files)
EXCLUDES="--exclude=/mnt --exclude=/proc --exclude=/sys --exclude=/dev \
    --exclude=/tmp --exclude=/run --exclude=/var/tmp --exclude=/var/lib/vz/images \
    --exclude=/var/lib/lxc --exclude=/var/lib/qemu"

# Start the backup process and log it
echo "Starting backup creation..." | tee -a "$LOG_FILE"

# Create the backup in /tmp on PVE, excluding certain directories
tar $EXCLUDES -czf "/tmp/$BACKUP_NAME" \
    /etc /root /home /var/lib/vz /etc/pve /var/lib/pve-cluster /boot/grub /etc/network \
    2>&1 | tee -a "$LOG_FILE"

# Check if the tar command was successful
if [ $? -eq 0 ]; then
    echo "Backup created successfully in /tmp/$BACKUP_NAME." | tee -a "$LOG_FILE"
else
    echo "Backup creation failed!" | tee -a "$LOG_FILE"
    exit 1
fi

# Transfer the backup file to OMV using rsync
echo "Transferring backup to OMV..." | tee -a "$LOG_FILE"
rsync -avz -e "ssh -i /root/.ssh/pve_to_omv_backup_ed25519" "/tmp/$BACKUP_NAME" pvesvc@192.168.x.x:"$BACKUP_DIR" 2>&1 | tee -a "$LOG_FILE"

# Check if the rsync command was successful
if [ $? -eq 0 ]; then
    echo "Backup transferred to OMV successfully." | tee -a "$LOG_FILE"
else
    echo "Backup transfer to OMV failed!" | tee -a "$LOG_FILE"
    exit 1
fi

# Clean up the temporary backup file from PVE
echo "Cleaning up temporary backup file from PVE..." | tee -a "$LOG_FILE"
rm -f "/tmp/$BACKUP_NAME"

# Check if the file was removed
if [ ! -f "/tmp/$BACKUP_NAME" ]; then
    echo "Temporary backup file successfully removed." | tee -a "$LOG_FILE"
else
    echo "Failed to remove temporary backup file!" | tee -a "$LOG_FILE"
    exit 1
fi

# Ensure that there are only 3 backups on OMV
echo "Checking and removing old backups on OMV..." | tee -a "$LOG_FILE"
BACKUPS_ON_OMV=$(ssh -i /root/.ssh/pve_to_omv_backup_ed25519 pvesvc@192.168.x.x "ls -1 $BACKUP_DIR | wc -l")

if [ "$BACKUPS_ON_OMV" -gt 3 ]; then
    # Remove the oldest backup if there are more than 3 backups
    OLD_BACKUPS=$(ssh -i /root/.ssh/pve_to_omv_backup_ed25519 pvesvc@192.168.x.x \
    "ls -t $BACKUP_DIR | tail -n +4")

    echo "Removing old backups on OMV..." | tee -a "$LOG_FILE"
    for BACKUP in $OLD_BACKUPS; do
        ssh -i /root/.ssh/pve_to_omv_backup_ed25519 pvesvc@192.168.x.x \
            "rm -f $BACKUP_DIR/$BACKUP" 2>&1 | tee -a "$LOG_FILE"
    done
else
    echo "Backup count on OMV is within limits. No cleanup needed." | tee -a "$LOG_FILE"
fi

echo "Backup process completed successfully." | tee -a "$LOG_FILE"
```

---

## Usage Notes

* Run this script as root or with appropriate permissions on the Proxmox VE host.
* SSH keys must be configured for passwordless access between PVE and OMV.
* Excludes VM disk and container data to avoid large bulky backups and focuses on OS and config files.
* Log file `/var/log/pve_backup.log` tracks backup creation, transfer, cleanup, and errors.
* Retains only 3 most recent backups on OMV to control storage usage.

---

## Workflow Summary

| Step                          | Description                                        |
| ----------------------------- | -------------------------------------------------- |
| 1. Create tarball backup      | Archive key system directories excluding VM images |
| 2. Transfer backup to OMV     | Use rsync over SSH with key authentication         |
| 3. Clean up local backup file | Delete temporary tarball after transfer            |
| 4. Remote backup retention    | Keep latest 3 backups on OMV, delete older         |

---
