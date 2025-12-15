This runbook outlines backup and recovery procedures for the OpenMediaVault (OMV) environment.

---

## Backup Summary

- **Backup Location:** `/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups`
- **Storage Pool:** backup_cifs share
- **Retention Policy:**
 - System configuration backups: Last backup only
 - Full image backups: Last backup only
- **Schedules:**
 - **Daily:** 2:10 PM - System configuration backup
 - **Monthly:** 3:00 AM - Full image backup

---

## Backup Services Overview

| Service | Backup Tool | Frequency | Location | Retention |
|---------|-------------|-----------|----------|-----------|
| OMV Configuration | `/usr/local/bin/backup-omv.sh` | Daily 2:10 PM | `backup_cifs/omv_backups` | Last only |
| OMV Full Image | `/usr/local/bin/full_omv_image.sh` | Monthly 3:00 AM | `backup_cifs/omv_backups/full_image_backups` | Last only |

---

## Backup Processes

### 1. System Configuration Backup (OS-level)

**Function:** Selective backup of critical OMV configuration files and directories.

- **Script Path:** `/usr/local/bin/backup-omv.sh`
- **Log File:** `/var/log/omv_backup.log`
- **Execution:** Daily via root cron (2:10 PM)
- **Method:** rsync with include/exclude patterns
- **Includes:**
 - `/boot/` and `/boot/efi/`
 - `/etc/` (complete directory tree)
 - `/etc/openmediavault/` (OMV configuration)
 - `/home/` (user directories)
 - `/root/` (root user files)
 - `/usr/local/` (local binaries/scripts)
 - `/var/lib/openmediavault/` (OMV data)
 - `/var/spool/cron/` (cron jobs)
 - `/var/log/` (system logs)
 - `/lib/udev/rules.d/` (udev rules)
 - `/etc/systemd/` (systemd configurations)

**Additional Outputs:**
- **Package List:** `manual_packages_TIMESTAMP.txt` - List of manually installed packages
- **OS Version:** `os_version_TIMESTAMP.txt` - System version information
- **Critical Files:** Individual copies of `/etc/fstab`, `/etc/openmediavault/config.xml`, authentication files

---

### 2. Full System Image Backup

**Function:** Complete filesystem image backup of root partition using fsarchiver.

- **Script Path:** `/usr/local/bin/full_omv_image.sh`
- **Log File:** `/var/log/omv_full_image_backup.log`
- **Execution:** Monthly via root cron (3:00 AM)
- **Method:** fsarchiver live backup
- **Target Partition:** `/dev/sdd2` (root filesystem)
- **Compression:** Multi-threaded (4 jobs)
- **Output Format:** `.fsa` (fsarchiver format)

---

## Restore Procedures

### System Configuration Restore

**Use Case:** Restore OMV configuration after system corruption or migration.

**Prepare target system:**

  - **Install fresh OMV on target hardware**
  - **Ensure same disk layout and mount points**

---

## Restore Summary
- **Backup Location:** `/srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups`
- **Storage Pool:** backup_cifs share
- **Recovery Types:**
 - Configuration restore: Partial system recovery
 - Full image restore: Complete system recovery
- **Prerequisites:**
 - **Configuration Restore:** Fresh OMV installation required
 - **Image Restore:** Boot from rescue media required

---

## Restore Services Overview
| Restore Type | Method | Use Case | Requirements | Recovery Time |
|--------------|--------|----------|--------------|---------------|
| Configuration | rsync + package reinstall | System corruption/migration | Fresh OMV install | 30-60 minutes |
| Full Image | fsarchiver restore | Hardware failure/complete loss | Rescue media boot | 2-4 hours |

---

## Restore Procedures

### 1. Configuration Restore (Partial System Recovery)
**Use Case:** Restore OMV configuration after system corruption or migration.

1. **Prepare target system:**
  ```bash
  # Install fresh OMV on target hardware
  # Ensure same disk layout and mount points
  ```

2. **Stop OMV Services:**
  ```bash
  systemctl stop openmediavault-*
  systemctl stop nginx
  systemctl stop php*-fpm
  ```

3. **Restore Configuration Files:**
  ```bash
  # Navigate to backup location
  cd /srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups

  # Find latest backup
  LATEST_BACKUP=$(ls -1dt os_backup_*/ | head -1)

  # Restore critical directories
  rsync -aAXHv "$LATEST_BACKUP"etc/ /etc/
  rsync -aAXHv "$LATEST_BACKUP"var/lib/openmediavault/ /var/lib/openmediavault/
  rsync -aAXHv "$LATEST_BACKUP"usr/local/ /usr/local/
  ```

4. **Restore Manually Installed Packages:**
  ```bash
  # Identify latest package list
  LATEST_PKG=$(ls -1t manual_packages_*.txt | head -1)

  # Update and reinstall
  apt update
  apt install $(cat "$LATEST_PKG")
  ```

5. **Restart Services:**
  ```bash
  systemctl start openmediavault-*
  omv-confgen nginx
  systemctl restart nginx
  ```

---

### 2. Full System Image Restore
**Use Case:** Complete system recovery after hardware failure.

1. **Prepare Target Hardware:**
 - Boot from OMV rescue media or any Linux live environment
 - Ensure fsarchiver is available

2. **Partition Target Disk:**
 - Create identical partition layout to original system
 - Ensure target partition (e.g., `/dev/sdd2`) exists

3. **Restore Filesystem Image:**
  ```bash
  # Navigate to image backup location
  cd /srv/dev-disk-by-uuid-ab75fcb7-21d3-4d7c-9f2d-257cf8fa074f/backup_cifs/omv_backups/full_image_backups

  # Find latest backup
  LATEST_IMAGE=$(ls -1t omv_full_backup_*.fsa | head -1)

  # Restore image to target partition
  fsarchiver restfs "$LATEST_IMAGE" id=0,dest=/dev/sdd2
  ```

4. **Update System Configuration:**
  ```bash
  # Mount restored filesystem
  mount /dev/sdd2 /mnt/target

  # Update /etc/fstab if UUIDs changed
  blkid >> /mnt/target/etc/fstab.new
  # Edit fstab.new manually and replace original when done
  ```

  ```bash
  # Prepare chroot environment
  mount --bind /dev /mnt/target/dev
  mount --bind /proc /mnt/target/proc
  mount --bind /sys /mnt/target/sys
  chroot /mnt/target

  # Install and update GRUB
  grub-install /dev/sdd
  update-grub
  exit
  ```

5. **Unmount and Reboot:**
  ```bash
  umount /mnt/target/dev /mnt/target/proc /mnt/target/sys
  umount /mnt/target
  reboot
  ```

---

## Post-Restore Validation

### System Status
**Verify core services are running:**
```bash
systemctl status openmediavault-*
systemctl status nginx
systemctl status samba
```

### Web Interface Access
- Access OMV Web GUI
- Confirm user authentication
- Check dashboard for system health

### Storage and Shares
- Verify filesystem mounts
- Test SMB/CIFS share accessibility
- Confirm shared folder permissions

### Network Configuration
- Verify IP configuration
- Test connectivity (e.g., ping, curl, nslookup)
- Confirm DNS resolution

### Log Review
**Check for errors and warnings:**
```bash
tail -f /var/log/daemon.log
tail -f /var/log/syslog
journalctl -u openmediavault-* --since "1 hour ago"
```

---

## Troubleshooting Common Issues

### Configuration Restore Issues
- **Service startup failures:** Check `/var/log/syslog` for dependency issues
- **Web GUI inaccessible:** Verify nginx configuration and SSL certificates
- **Share access problems:** Confirm user permissions and SMB service status

### Full Image Restore Issues
- **Boot failures:** Verify GRUB installation and `/etc/fstab` UUID mappings
- **Partition size mismatches:** Ensure target partition is same size or larger
- **Network configuration:** Update IP settings if hardware MAC addresses changed

### General Recovery Tips
- Always test restore procedures in non-production environment
- Document any configuration changes made during restore
- Verify backup integrity before attempting restore
- Keep rescue media and tools readily available
