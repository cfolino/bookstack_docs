Backup Summary
--------------

*   **Backup Target:** Proxmox Backup Server datastore

*   **Backup Method:** VM snapshots via PBS jobs

*   **Schedule:** Daily at 10:00 AM

*   **Retention Policy:** Last 7 snapshots retained per VM

*   **VM Discovery:** Automatic - all VMs are auto-added to snapshot jobs


Backup Process
--------------

### VM Snapshot Jobs

**Function:** Automated daily snapshots of all virtual machines in the Proxmox VE cluster.

*   **Execution Time:** 10:00 AM daily

*   **Backup Type:** VM snapshots (consistent point-in-time copies)

*   **Target Storage:** PBS datastore

*   **Auto-Discovery:** VMs are automatically added to backup jobs upon creation

*   **Retention:** Rolling 7-day retention (oldest snapshot automatically pruned)**Key Features:**

*   **Crash-consistent snapshots** for running VMs

*   **Application-consistent snapshots** when QEMU guest agent is available

*   **Incremental backups** after initial full snapshot

*   **Deduplication** and compression at PBS level

*   **Verification** jobs run automatically post-backup


Backup Job Configuration
------------------------

### Default VM Backup Job Settings

```bash
# Job Schedule
Schedule: daily 10:00
Mode: snapshot
Compression: zstd (fast)
Protected: false
```

### Retention Settings

```bash
keep-last: 7
keep-daily: 0
keep-weekly: 0
keep-monthly: 0
keep-yearly: 0
```

Monitoring and Validation
-------------------------

### Daily Verification Tasks

1.  **PBS Web UI Dashboard** - Check job completion status

2.  **Email Notifications** - Review backup job reports (if configured)

3.  **Log Review** - Examine PBS task logs for errors:

```bash
journalctl -u
proxmox-backup.service -f
```

### Key Metrics to Monitor

*   **Backup Duration** - Normal completion time vs. outliers

*   **Storage Usage** - Datastore capacity and growth trends

*   **Failed Jobs** - VMs that consistently fail backup

*   **VM Agent Status** - QEMU guest agent availability for consistent snapshots


Restore Procedures
------------------

### Single VM Restore

1.  Access PBS Web UI or use proxmox-backup-client

2.  Navigate to datastore and select VM backup

3.  Choose restore point (snapshot timestamp)

4.  Restore options:

    *   **Full VM restore** to original or new VMID

    *   **Individual disk restore** for selective recovery

    *   **File-level restore** via PBS file browser
