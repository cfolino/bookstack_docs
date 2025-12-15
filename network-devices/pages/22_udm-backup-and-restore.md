This section documents the automated backup and manual restore process for the UniFi Dream Machine (UDM).

---

## Backup Overview

- **Script Path:** `/home/udmsvc/pull-udm-backups.sh`
- **Log File:** `/home/udmsvc/backup_log.txt`
- **Execution:** Daily via cron (12 PM)
- **Retention:** All backups retained
- **Destination:**
  `/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/udm_backups`

---

## Restore Procedure

1. **Identify the latest backup archive** (e.g., `autobackup_xxxx-xx-xx.unf`) in:

    ```bash
    /srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/udm_backups/
    ```

2. **Transfer the backup to the UDM** using `scp`:

    ```bash
    scp -i /home/udmsvc/.ssh/omv_to_udm_backup autobackup_xxxx-xx-xx.unf root@192.168.10.1:/mnt/data/unifi/backup/
    ```

3. **SSH into the UDM**:

    ```bash
    ssh -i /home/udmsvc/.ssh/omv_to_udm_backup root@192.168.10.1
    ```

4. **Restore via the UniFi Controller web interface**:

    - Open the UniFi Controller in your browser.
    - Go to **Settings → System → Backup**.
    - Choose the uploaded `.unf` file.
    - Click **Restore Backup** and follow the prompts.

5. **Reboot the UDM** if not automatically restarted:

    ```bash
    reboot
    ```

---

✅ **Post-Restore Validation:**

- Confirm UniFi Controller accessibility via web UI.
- Validate site, device, and user settings were restored.
- Confirm adoption status of UniFi devices.
