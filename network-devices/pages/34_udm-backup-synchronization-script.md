## Purpose

This script synchronizes UniFi Dream Machine (UDM) automatic backups from the UDM device to an OpenMediaVault (OMV) server using `rsync` over SSH. It efficiently copies only new or changed backup files, skipping files already present on OMV.

---

## Overview

* Uses `rsync` with SSH key authentication for secure file transfer from UDM to OMV.
* Syncs UDM’s automated backup directory to a persistent backup folder on OMV.
* Appends a timestamped entry to a local log file after each successful sync.

---

## Configuration Variables

| Variable         | Description                                                                     |
| ---------------- | ------------------------------------------------------------------------------- |
| `UDM_IP`         | IP address of the UDM device                                                    |
| `UDM_BACKUP_DIR` | UDM automatic backup source directory                                           |
| `OMV_BACKUP_DIR` | Destination directory on OMV server                                             |
| SSH key          | Private key for authentication located at `/home/udmsvc/.ssh/omv_to_udm_backup` |

---

## Script Content

```bash
#!/bin/bash

# UDM details
UDM_IP="192.168.10.1"              # UDM IP address
UDM_BACKUP_DIR="/data/unifi/data/backup/autobackup"  # UDM backup directory
OMV_BACKUP_DIR="/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/udm_backups"  # OMV backup directory

# Use rsync to pull the contents from UDM and skip already copied files
rsync -avz --ignore-existing -e "ssh -i /home/udmsvc/.ssh/omv_to_udm_backup" root@$UDM_IP:$UDM_BACKUP_DIR/ "$OMV_BACKUP_DIR"

# Log the backup time for reference
DATE=$(date '+%Y-%m-%d_%H-%M-%S')
echo "Backup completed at $DATE" >> /home/udmsvc/backup_log.txt
```

---

## Usage Notes

* Run as the `udmsvc` user or another user with SSH key access to UDM and write permissions on OMV backup directory.
* SSH key `/home/udmsvc/.ssh/omv_to_udm_backup` must be set up for passwordless authentication.
* Logs the timestamp of the last backup sync to `/home/udmsvc/backup_log.txt`.
* Efficiently transfers only new or changed files to minimize bandwidth and storage.

---

## Workflow Summary

| Step                     | Description                         |
| ------------------------ | ----------------------------------- |
| 1. Connect to UDM        | Using SSH and rsync to UDM IP       |
| 2. Sync backup directory | Copy new backups to OMV destination |
| 3. Log completion time   | Append timestamp to local log file  |

---
