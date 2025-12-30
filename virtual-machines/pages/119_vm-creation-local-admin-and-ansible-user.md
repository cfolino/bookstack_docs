## Overview

When creating a new virtual machine, the **local administrator account must be named `cfolinoadmin`**.

This account exists for two reasons only:
- To provide **break-glass / emergency access**
- To allow **initial automation bootstrap** so the `ansible` user can be created

After bootstrap is complete, **all automation and system management is performed using the `ansible` user**.
The `cfolinoadmin` account remains on the system but is no longer used by Ansible or Semaphore.

---

## Why the Local Admin Must Be `cfolinoadmin`

Using a standardized local admin account ensures:

- A consistent, known admin user on every VM
- Password-based access is available only when needed
- No passwords are stored in Git or hard-coded in automation
- Semaphore can temporarily use the password to create the `ansible` user
- A reliable break-glass account exists if SSH keys or automation fail

---

## How the Password Is Used

- The password for `cfolinoadmin` is set manually when the VM is created
- That same password is then supplied **temporarily** to the Semaphore **Bootstrap** job as a variable
- Ansible uses this password one time to:
  - SSH into the VM as `cfolinoadmin`
  - Run sudo
  - Create and configure the `ansible` user
- After bootstrap:
  - The password variable is removed from Semaphore
  - Automation switches permanently to SSH key authentication as `ansible`

The password is never committed to source control and never reused after bootstrap.

---

## Required VM Creation Standard

When creating a new VM:

- Create a local user named: cfolinoadmin
- Add the user to the sudo group (password required)
- Enable SSH password authentication
- Use a strong, unique password
- Do NOT create the ansible user manually

---

## Bootstrap Flow Summary

1. VM is created with local admin user `cfolinoadmin`
2. VM is added to the Ansible `bootstrap` inventory group
3. Semaphore Bootstrap job is run with a temporary password variable
4. Ansible creates and configures the `ansible` user
5. VM is removed from the `bootstrap` group
6. The temporary password variable is deleted
7. VM is now managed exclusively via `ansible` and SSH keys

---

## Verification Commands

After the bootstrap job completes, use the following commands to verify the system state.

### Verify the ansible user exists (run on the VM)
```
id ansible
```
### Verify ansible can log in and use sudo without a password
```
ssh ansible@<vm_name> whoami
ssh ansible@<vm_name> sudo -n true
```
### Verify Ansible connectivity (run from the Ansible control node)
```
ansible <vm_name> -m ping
ansible <vm_name> -m command -a "id" -b
```
All commands should complete successfully and must not prompt for a password.

---

## End State (What “Correct” Looks Like)

- The `cfolinoadmin` account exists and can be used for break-glass access
- The `ansible` account exists and has passwordless sudo
- SSH key authentication works for `ansible`
- The VM is no longer in the `bootstrap` inventory group
- No Semaphore job contains the bootstrap password variable

At this point, the VM is fully onboarded and ready for normal automation.
