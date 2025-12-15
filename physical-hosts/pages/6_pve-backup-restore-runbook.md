This runbook outlines backup and recovery procedures for the Proxmox VE (PVE) environment.

---

## Backup Summary

- **Backup Location on OMV:** `/srv/dev-disk-by-uuid-<uuid-redacted>/backup_cifs/pve_backups`
- **Primary Offload Target:** OMV server `<internal-host>` (192.168.15.225)
- **Retention Policy:**
  - System backups: Last 3 backups
  - ZFS backups: Last full backup only (monthly rotation)
- **Schedules:**
  - **Daily:** 2 PM - Full system configuration backup
  - **Monthly:** 4 AM - Full ZFS pool backup

---

## Backup Processes

### 1. System Configuration Backup (OS-level)

**Function:** Backup PVE configuration files excluding VM disks.

- **Script Path:** `/root/scripts/pve_os_backup.sh`
- **Log File:** `/var/log/pve_backup.log`
- **Execution:** Daily via cron (2 PM)

---

### 2. Full ZFS Pool Backup

**Function:** Capture full state of ZFS storage pool.

- **Script Path:** `/root/scripts/full_rpool_zfs_backup.sh`
- **Log File:** `/var/log/full_rpool_backup.log`
- **Execution:** Monthly via cron (4 AM)
- **Retention:** Only last backup is retained.

---

## Restore Procedures

### System Restore (OS)

1. Boot Live Linux environment on replacement hardware.
2. Partition target disk.
3. Mount target filesystem.
4. Extract system backup archive:
    ```bash
    tar -xzf /path/to/pve_backup_TIMESTAMP.tar.gz -C /mnt/target
    ```
5. Reinstall GRUB:
    ```bash
    mount --bind /dev /mnt/target/dev
    mount --bind /proc /mnt/target/proc
    mount --bind /sys /mnt/target/sys
    chroot /mnt/target
    grub-install /dev/sdX
    update-grub
    exit
    ```
6. Verify `/etc/fstab` and network configuration.

---

### ZFS Restore

1. Transfer ZFS backup file from OMV.
2. Decompress:
    ```bash
    zstd -d < rpool_full_TIMESTAMP.zfs.zst > rpool_full_TIMESTAMP.zfs
    ```
3. Receive ZFS stream:
    ```bash
    zfs receive -Fdu rpool < rpool_full_TIMESTAMP.zfs
    ```

---

## Validation

- Confirm `pvecm status` for cluster sync.
- Validate VM status and accessibility.
- Review logs for errors.
