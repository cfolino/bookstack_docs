## Overview
This procedure outlines the automated rolling maintenance for the cfolino-cluster (Dell R630 and Optiplex nodes). To ensure 100% uptime for services and maintain Corosync quorum, this workflow utilizes a Serial Migration strategy. Nodes are patched and rebooted one-by-one. Running workloads (VMs/LXCs) are live-migrated to a neighboring node before the host initiates a reboot.

---

## Infrastructure Components
* Orchestrator: Ansible / Semaphore
* Target Group: proxmox (pve, pve2, pve3)
* Hypervisor: Proxmox VE (PVE)
* Storage: ZFS Local Replication / Shared Storage
* Safety Mechanism: serial: 1 execution

---

## Logical Workflow

1. Pre-flight Check: Verifies the host is a Proxmox node and checks for ZFS health and Cluster Quorum (Quorate: Yes).
2. Patching: Performs a dist-upgrade using ionice -c 3 to ensure I/O priority remains with running VMs.
3. Workload Migration:
    * Sets HA state to maintenance to prevent automatic failback.
    * Identifies running VMs on the host.
    * Calculates a neighbor target: pve -> pve2, pve2 -> pve3, pve3 -> pve.
    * Verifies target node availability before migration.
    * Executes pvenode migrateall with --with-local-disks 1 and --maxworkers 1.
4. Decoupled Reboot: Uses systemd-run to schedule a reboot 15 seconds after the Ansible task completes, preventing SSH hang-ups.
5. Reconnection and Verification: Wait for the node to return to the network, verify Corosync quorum, and clear HA maintenance mode before proceeding to the next node.

---

## Configuration

### Playbook: playbooks/pve_patch.yml

    ---
    - name: Patch and reboot Proxmox VE cluster
      hosts: proxmox
      serial: 1
      gather_facts: yes
      become: yes

      roles:
        - role: pve_patch
          vars:
            pve_patch_dry_run: false
            pve_patch_do_reboot: true
            pve_patch_enable_notify: true

### Key Task: reboot.yml

    - name: PVE | Set HA to maintenance mode
      ansible.builtin.command: "ha-manager set-state --node {{ inventory_hostname }} maintenance"
      when: not pve_patch_dry_run
      ignore_errors: yes

    - name: PVE | Determine migration target node
      ansible.builtin.set_fact:
        pve_migration_target: "{{ 'pve2' if inventory_hostname == 'pve' else ('pve3' if inventory_hostname == 'pve2' else 'pve') }}"
      when: running_vms.stdout != ""

    - name: PVE | Ensure migration target is online
      ansible.builtin.wait_for:
        host: "{{ hostvars[pve_migration_target]['ansible_host'] }}"
        port: 22
        timeout: 300
      when: running_vms.stdout != ""

    - name: PVE | Evacuate VMs to {{ pve_migration_target }}
      ansible.builtin.command: "pvenode migrateall {{ pve_migration_target }} --with-local-disks 1 --maxworkers 1"
      async: 1800
      poll: 10
      when:
        - not pve_patch_dry_run
        - running_vms.stdout != ""

    - name: PVE | Schedule reboot
      ansible.builtin.command: systemd-run --on-active=15 /sbin/reboot
      when: not pve_patch_dry_run

    - name: PVE | Wait for node to return
      ansible.builtin.wait_for_connection:
        delay: 60
        timeout: 600

    - name: PVE | Clear HA maintenance mode
      ansible.builtin.command: "ha-manager set-state --node {{ inventory_hostname }} started"
      when: not pve_patch_dry_run
      ignore_errors: yes

---

## Execution and Troubleshooting

### CLI Commands

Dry Run (Test Logic Only):
   ```
ansible-playbook -i inventory/hosts.yml playbooks/pve_patch.yml -e "pve_patch_dry_run=true"
```
Live Run:
```
ansible-playbook -i inventory/hosts.yml playbooks/pve_patch.yml
```
### Troubleshooting
* Migration Fails: Check for local ISOs attached to VMs or specific hardware passthrough (USB/GPU) that prevents live migration.
* Corosync Issues: If a node reboots and the next node fails pre-check, run pvecm status. Ensure at least 2 out of 3 nodes are online to maintain quorum.
* Stuck Reboot: If a node does not return within 10 minutes, use the iDRAC (R630) or physical console to check for GRUB menu hangs or ZFS mount issues.
* HA Fighting Migration: Ensure the ha-manager set-state maintenance command executed successfully.

### Verification Logic (Under-the-Hood)
The pre-check uses a regex match to handle Proxmox specific table formatting:
    failed_when: cluster_status.stdout is not search('Quorate:\s+Yes')
This ensures that hidden whitespace or terminal characters do not cause false failures in the automation pipeline.
