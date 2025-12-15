This page documents the **automated** pipeline used to export internal BookStack documentation, sanitize sensitive data, and publish a public-safe mirror to GitHub.

The internal BookStack instance remains the authoritative source of truth.
GitHub is treated as a read-only mirror for portfolio and reference purposes.

---

## Pipeline Overview

The pipeline is intentionally split into clear, single-responsibility stages:

1. Export – Lossless extraction from BookStack
2. Sanitize – Deterministic redaction of sensitive data
3. Publish – Version-controlled push to GitHub with safety gates
4. Verify – Continuous integration leak scanning in GitHub

All steps are automated to prevent accidental disclosure.

---

## Directory Layout

```text
/home/bookstack/
├── bookstack-export.sh
├── sanitize-export.sh
├── publish-bookstack-export.sh
├── bookstack_export_YYYYMMDD_HHMMSS/
│   └── (raw BookStack export)
└── bookstack_export_sanitized/
    └── (sanitized GitHub mirror)
```

- Timestamped export directories are immutable
- bookstack_export_sanitized is the only directory under Git version control

---

## Step 1: Export (Lossless)

Script: `bookstack-export.sh`

This script:
- Uses the BookStack API
- Exports books, chapters, and pages
- Preserves Markdown exactly as authored
- Writes output to a timestamped directory

No sanitization occurs at this stage.

```bash
./bookstack-export.sh
```

---

## Step 2: Sanitize (Redaction)

Script: `sanitize-export.sh`

This script performs deterministic redaction of sensitive data.

### Redacted content
- Internal domains
- Internal email addresses
- RFC1918 IP addresses
- Disk UUIDs and UUID-based mount paths

### Preserved content
- Directory structure
- Script logic
- Public keys
- Standard Linux paths

### Example redactions

```text
192.168.10.44              → 192.168.x.x
omv.cfolino.com            → omv.internal
alerts@cfolino.com         → <redacted-email>
dev-disk-by-uuid-<uuid>    → dev-disk-by-uuid-<uuid-redacted>
```

---

## Step 3: Publish (Authoritative)

Script: `publish-bookstack-export.sh`

This is the only script allowed to push content to GitHub.

It performs the following:
1. Rsync latest export into the sanitized directory
2. Run the sanitizer automatically
3. Enforce safety gates
4. Commit changes
5. Force-update the GitHub mirror

```bash
./publish-bookstack-export.sh
```

---

## Step 4: CI Leak Scanning (GitHub)

A GitHub Actions workflow runs on every push and pull request to validate that no sensitive data is present.

### What the CI scan enforces
- No private key material
- No internal domains
- No internal email addresses (redacted placeholders allowed)
- No raw RFC1918 IP addresses
- No cloud API keys or bearer tokens

### Purpose

This provides a second, external safety net in addition to local sanitization and publish-time checks.

Even if local safeguards are bypassed or modified, GitHub will refuse unsafe content.

---

## Repository README Handling

README.md is repository-owned metadata and is excluded from export synchronization.

It explains that the repository is a mirror and persists across exports.

---

## Normal Usage Workflow

```bash
./bookstack-export.sh
./publish-bookstack-export.sh
```

CI verification runs automatically after every push.

---

## Summary

This pipeline ensures:
- Internal documentation remains complete and accurate
- Public documentation is safe by construction
- Redaction is automated and verifiable
- External CI enforces an additional safety boundary

The separation between authoring, sanitization, publishing, and verification is intentional.
