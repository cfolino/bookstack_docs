This system ensures your VMs only start after your ZFS pool is confirmed healthy, and allows you to configure VM boot order and delay to avoid boot storms or dependency issues.

---

## File Locations

| Script/Unit                   | Path                                           |
| ----------------------------- | ---------------------------------------------- |
| VM Autostart Config Script    | `/root/scripts/add-vm-to-autostart.sh`            |
| ZFS-Aware Startup Wait Script | `/root/scripts/wait-for-zfs-and-start-vms.sh`  |
| systemd Unit for Wait Logic   | `/etc/systemd/system/zfs-vm-autostart.service` |

---

##  `add-vm-to-autostart.sh` — Configure VM Boot Order & Delay

###  Purpose

Set a Proxmox VM to autostart after host reboot, with a custom order and delay.

---

###  Example Usage

```bash
./add-vm-to-autostart.sh 101 1 15
```

###  Argument Breakdown

| Argument | Meaning                                                                      |
| -------- | ---------------------------------------------------------------------------- |
| `101`    | VMID of the VM (from `qm list`)                                              |
| `1`      | Startup **order** (lower boots first). VMs with same order boot in parallel. |
| `15`     | Startup **delay** (seconds) **after** the previous VM has started            |

---

### Ordering Scheme

| Role                        | Order | Delay (up) |
|-----------------------------|-------|------------|
| Core infra (DNS, AD)        | 1     | 10–15s     |
| Storage servers (NFS/iSCSI) | 2     | 10–30s     |
| App VMs (BookStack, Grafana)| 3     | 15–30s     |
| Monitoring or backups       | 4     | 0–10s      |

---

###  Script Content

```bash
#!/bin/bash
set -euo pipefail

if [ $# -lt 1 ]; then
  echo "Usage: $0 <VMID> [order (default=99)] [delay_seconds (default=10)]"
  exit 1
fi

VMID="$1"
ORDER="${2:-99}"
DELAY="${3:-10}"

echo "Setting VMID=$VMID to autostart with order=$ORDER and delay=${DELAY}s..."

qm set "$VMID" --onboot 1
qm set "$VMID" --startup order="$ORDER",up="$DELAY"

echo "Verification of VM config:"
qm config "$VMID" | grep -E "onboot|startup"
```

---

###  Verification

```bash
qm config 101 | grep -E "onboot|startup"
```

Example output:

```
onboot: 1
startup: order=1,up=15
```

---

##  `wait-for-zfs-and-start-vms.sh` — ZFS Pool Health Wait

###  Purpose

Delays VM startup until your ZFS pool (e.g., `main`) is confirmed `ONLINE`, preventing boot-time failures.

---

###  Used By

Systemd unit: `/etc/systemd/system/zfs-vm-autostart.service`

---

###  Process Flow

1. Checks ZFS pool health using `zpool list`.
2. Retries (with backoff) until the pool is ONLINE or retries max out.
3. Logs status and errors using `logger`.
4. Signals `systemd` with `systemd-notify --ready` if successful.

---

###  Script Content

```bash
#!/bin/bash
set -euo pipefail

POOL="main"
MAX_RETRIES=12
INITIAL_WAIT=5
MAX_WAIT=30

log() {
    logger -t zfs-vm-autostart "$1"
    echo "$(date '+%F %T') $1"
}

notify_ready() {
    if [[ -n "${NOTIFY_SOCKET:-}" ]]; then
        systemd-notify --ready
    fi
}

log "Starting ZFS pool availability check for '$POOL'."

for (( retry=1; retry<=MAX_RETRIES; retry++ )); do
    if zpool list -H -o health "$POOL" 2>/dev/null | grep -q "ONLINE"; then
        log "ZFS pool '$POOL' is ONLINE."
        notify_ready
        exit 0
    fi

    WAIT_TIME=$(( INITIAL_WAIT * retry ))
    [[ $WAIT_TIME -gt $MAX_WAIT ]] && WAIT_TIME=$MAX_WAIT

    log "Attempt $retry: ZFS pool '$POOL' not online yet. Waiting $WAIT_TIME seconds..."
    sleep "$WAIT_TIME"
done

log "ERROR: ZFS pool '$POOL' did not become available after $MAX_RETRIES attempts."
notify_ready
exit 1
```

---

##  systemd Unit: `zfs-vm-autostart.service`

###  Unit File Content

```ini
[Unit]
Description=Wait for ZFS pool to be online before starting Proxmox guests
After=zfs-import.target
Requires=zfs-import.target
ConditionPathExists=/root/scripts/wait-for-zfs-and-start-vms.sh

[Service]
Type=notify
ExecStart=/root/scripts/wait-for-zfs-and-start-vms.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

###  Activation Steps

```bash
chmod +x /root/scripts/wait-for-zfs-and-start-vms.sh
systemctl daemon-reload
systemctl enable zfs-vm-autostart.service
```

---

##  Workflow Summary

| Phase         | Action                                               |
| ------------- | ---------------------------------------------------- |
| System boot   | `zfs-vm-autostart.service` runs                      |
| Waits for ZFS | `wait-for-zfs-and-start-vms.sh` retries health check |
| ZFS Ready     | systemd marked ready, VM startup allowed             |
| VM boot order | Controlled by `set-vm-autostart.sh` config per VM    |

---

##  Notes

* This setup is fully **reversible**: disable or remove the service and unset startup flags.
* Extendable: Add more VMs using `set-vm-autostart.sh` anytime.
* Safe: Avoids race conditions with disk not mounted when VMs boot.

---
