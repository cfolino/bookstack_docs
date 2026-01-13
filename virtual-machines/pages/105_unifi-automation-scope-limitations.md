---

This page documents the **intentional scope**, **control boundaries**, and **risk constraints** applied to UniFi automation within the homelab.

UniFi automation is treated as **situational and constrained**, not foundational infrastructure automation.

---

## Automation Philosophy for UniFi

UniFi devices occupy a unique position:

- They are **network control plane components**
- They affect **all downstream systems**
- They are **stateful and controller-driven**
- They are often impacted by firmware and API changes

For these reasons, UniFi automation is designed to be:

- Explicit
- Narrow in scope
- Operator-initiated
- Non-persistent

Automation is used to **assist**, not to continuously manage.

---

## What Is Automated

UniFi automation currently supports:

- Device reachability checks
- Uptime verification
- Targeted reboots
- Controller API interaction
- UDM-specific reboot workflows

These actions are invoked **on demand**, not on schedules.

---

## What Is Intentionally NOT Automated

The following are **explicitly excluded** from automation:

- Configuration drift correction
- Policy enforcement
- Firewall rule management
- VLAN changes
- Firmware upgrades
- Continuous state reconciliation

Those actions remain **manual and deliberate** via the UniFi UI.

---

## Roles & Playbooks Involved

### Roles

```
roles/unifi_reboot
roles/unifi_uptime_check
```

### Playbooks

```
unifi_reboot_devices.yml
unifi_reboot_udm.yml
unifi_uptime_check.yml
unifi_preflight_check.yml
unifi_debug_*.yml
```

These playbooks are **tools**, not background services.

---

## API-Driven Control Model

UniFi automation uses:

- UniFi Controller API
- HTTPS requests
- Session-based authentication

There is **no SSH-based control** of UniFi devices.

This ensures:
- Controller remains authoritative
- Device state stays consistent
- No out-of-band mutations occur

---

## Certificate Validation Behavior

UniFi API interactions intentionally use:

```
validate_certs: false
```

This is a **conscious decision**, not an oversight.

### Why Certificate Validation Is Disabled

- UniFi controllers often use self-signed certs
- Internal CA trust is not consistently supported
- API endpoints change across versions
- Certificate rotation can break automation unexpectedly

The automation assumes:
- Traffic remains inside trusted VLANs
- The controller endpoint is known and controlled

Security is enforced by **network boundaries**, not TLS validation.

---

## Reboot Automation Strategy

Reboots are treated as **exceptional actions**, not routine operations.

Automation supports:
- Individual device reboots
- Group reboots
- UDM-specific reboot sequencing

Reboots are never:
- Chained automatically
- Triggered by monitoring
- Coupled to patch cycles

This avoids network-wide outages caused by automation errors.

---

## UDM Special Handling

The UniFi Dream Machine (UDM):

- Hosts the controller
- Routes traffic
- Terminates VPNs
- Acts as a single point of failure

UDM automation is therefore:
- Isolated to dedicated playbooks
- Manually executed
- Verbose in output
- Never combined with other operations

---

## Debug & Inspection Playbooks

Files prefixed with `debug_` exist to:

- Inspect API responses
- Validate endpoints
- Confirm authentication
- Test controller behavior after upgrades

These are **diagnostic tools**, not operational workflows.

They are intentionally undocumented as runbooks.

---

## Failure & Risk Containment

UniFi automation is designed to fail safely:

- No retries that could loop
- No cascading actions
- No state enforcement
- No unattended changes

If an API call fails:
- The playbook exits
- No further action is taken
- Manual intervention is required

---

## Why This Boundary Exists

Automating network control systems aggressively introduces:

- High blast radius
- Difficult rollback
- Poor observability
- Vendor-driven breakage

By constraining automation, the homelab remains:
- Stable
- Predictable
- Recoverable

---

## Summary

- UniFi automation is **situational**
- API-driven, not SSH-driven
- Certificate validation is intentionally disabled
- Reboots are manual and explicit
- Configuration remains human-controlled

UniFi automation exists to **assist operators**, not replace them.

---

## Related Pages

- Inventory & Grouping Strategy
- Patch & Reboot Orchestration
- Proxmox-Driven VM Lifecycle Control
- Automation Email Architecture
