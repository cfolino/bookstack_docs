##  `pihole-update-sync.sh` — Gravity Update with Timestamped Logging and Sync Trigger

###  Purpose

Automates updating the Pi-hole gravity database with detailed timestamped logs, then triggers a sync script to replicate configuration to a backup Pi-hole server.

---

###  Overview

* Runs `pihole -g` to update gravity.
* Logs output with timestamps to `/var/log/pihole-update-sync.log`.
* Calls the sync script `/root/scripts/pihole-sync.sh` after the update completes.

---

###  Script Content

```bash
#!/bin/bash
# Update Pi-hole gravity with timestamped logging

# Redirect all stdout and stderr through awk for timestamping
exec > >(awk '{ print strftime("[%Y-%m-%d %H:%M:%S]"), $0 }' >> /var/log/pihole-update-sync.log) 2>&1

echo "Starting Pi-hole gravity update"

/usr/local/bin/pihole -g

echo "Gravity update complete, starting sync"

# Run your sync script
/root/scripts/pihole-sync.sh

echo "Sync complete"
```

---

###  Usage

Typically run as a cron job (e.g., weekly Wednesday 9 AM):

```cron
0 9 * * 3 /root/scripts/pihole-update-sync.sh
```

---

##  `pihole-sync.sh` — Rsync Pi-hole Config to Backup Server and Restart Service

###  Purpose

Synchronizes Pi-hole configuration files to a backup Pi-hole node securely via SSH and restarts the Pi-hole service on the backup to apply changes.

---

###  Overview

* Uses `rsync` over SSH with a dedicated key.
* Syncs `/etc/pihole/` and `/etc/dnsmasq.d/` directories.
* Restarts `pihole-FTL` service on the backup node.

---

### Important Variables

| Variable  | Description                      |
| --------- | -------------------------------- |
| `REMOTE`  | Backup Pi-hole host (user\@host) |
| `SSH_KEY` | SSH private key path for auth    |

---

###  Script Content

```bash
#!/bin/bash

# Set variables
REMOTE=pihole@pihole-backup
SSH_KEY=/root/.ssh/id_ed25519_gravitysync

# Sync Pi-hole configuration files
rsync -e "ssh -i $SSH_KEY" -avz --delete /etc/pihole/ "$REMOTE:/etc/pihole/"
rsync -e "ssh -i $SSH_KEY" -avz --delete /etc/dnsmasq.d/ "$REMOTE:/etc/dnsmasq.d/"

# Restart Pi-hole service on the backup node
ssh -i "$SSH_KEY" "$REMOTE" "sudo /usr/bin/systemctl restart pihole-FTL"
```

---

###  Usage Notes

* Ensure SSH key-based authentication is set up between primary and backup nodes.
* Confirm `pihole-FTL` service name matches your backup system.
* Run manually or from the gravity update script as a follow-up step.

---

##  Workflow Summary

| Step                         | Description                                                             |
| ---------------------------- | ----------------------------------------------------------------------- |
| 1. Gravity update            | `pihole-update-sync.sh` updates gravity database with timestamped logs. |
| 2. Configuration sync        | On success, triggers `pihole-sync.sh` to sync configs to backup.        |
| 3. Service restart on backup | Backup Pi-hole restarts `pihole-FTL` to apply new configs.              |

---
