## Purpose

Deploy the standard human admin user `cfolino` on VMs using Ansible + Semaphore.

This user provides:
- SSH key–based admin access
- Passwordless sudo
- Separation from service accounts (e.g. `ansible`)

---

## Semaphore Job

**Job name:**
```
Add Admin User – cfolino
```
**Playbook:**
```
admin-user.yml
```
**Execution:**
```
Manual only
```
---

## Required: Job Scoping

This job **must be scoped before execution**.

Use the **Limit** field in Semaphore to control where the admin user is created.

**Examples**

Single host
```
bookstack
````
Multiple hosts
```
bookstack, grafana
```
Inventory group
```
ubuntu_vms
```
**Do not run this job without a limit unless intentionally targeting all hosts.**

---

## Verification

After execution, verify access:
```
ssh cfolino@<host>
sudo -l
```
Expected:
- SSH login via key
- NOPASSWD sudo access

---

## Notes

- Safe to re-run
- Does not modify service accounts
- Intended for human administration only
