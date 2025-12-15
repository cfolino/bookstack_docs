This page documents the automated pipeline used to export internal BookStack documentation, sanitize sensitive data, and publish a **public-safe mirror** to GitHub.

The internal BookStack instance remains the **authoritative source of truth**.
GitHub is treated as a **read-only mirror** for portfolio and reference purposes.

---

## Pipeline Overview

The pipeline is intentionally split into clear, single-responsibility stages:

1. **Export** – Lossless extraction from BookStack
2. **Sanitize** – Deterministic redaction of sensitive data
3. **Publish** – Version-controlled push to GitHub with safety gates

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

- Timestamped export directories are **immutable**
- `bookstack_export_sanitized` is the **only** directory under Git version control

---

## Step 1: Export (Lossless)

**Script:** `bookstack-export.sh`

This script:
- Uses the BookStack API
- Exports books, chapters, and pages
- Preserves Markdown exactly as authored
- Writes output to a timestamped directory

**Important design choice:**
No sanitization or redaction happens at this stage. The export must remain a faithful internal copy.

```bash
./bookstack-export.sh
```

---

## Step 2: Sanitize (Redaction)

**Script:** `sanitize-export.sh`

This script operates only on the sanitized mirror directory and performs **deterministic redaction**.

### What is redacted
- Internal domains
- Internal email addresses
- RFC1918 IP addresses
- Disk UUIDs and UUID-based mount paths

### What is preserved
- Directory structure
- Script logic
- Standard Linux paths
- Public keys (never private keys)

### Example Redactions

```text
192.168.x.x              → 192.168.x.x
omv.internal            → omv.internal
alerts@<redacted-email>         → <redacted-email>
dev-disk-by-uuid-<uuid>    → dev-disk-by-uuid-<uuid-redacted>
```

The sanitizer is:
- Idempotent
- Safe to run multiple times
- Designed for auditability via `git diff`

---

## Step 3: Publish (Authoritative)

**Script:** `publish-bookstack-export.sh`

This is the **only script that should ever push to GitHub**.

It performs the following steps in order:

1. Rsync latest export into the sanitized directory
2. Run the sanitizer automatically
3. Enforce safety gates (fail if anything unsafe remains)
4. Commit changes to Git
5. Force-update the GitHub mirror

```bash
./publish-bookstack-export.sh
```

---

## Safety Gates

Publishing is aborted if any of the following are detected:
- Internal domains
- Internal email addresses
- Unsanitized RFC1918 IP addresses

This guarantees unsafe content **cannot be pushed**, even by mistake.

---

## Repository README Handling

`README.md` is treated as **repository-owned**, not export-owned.

- It explains that the repository is a mirror
- It is excluded from rsync deletion
- It persists across exports

This keeps contextual metadata separate from documentation content.

---

## GitHub Repository Model

- GitHub is a **one-way mirror**
- No direct edits are made in GitHub
- History may be force-updated intentionally
- Sanitized content only

This model avoids drift and ensures consistency.

---

## Normal Usage Workflow

```bash
./bookstack-export.sh
./publish-bookstack-export.sh
```

No manual sanitization steps are required.

---

## Design Goals

- Preserve internal documentation accuracy
- Prevent accidental data disclosure
- Enable public sharing without risk
- Maintain clean version history
- Keep the system simple and auditable

---

## Summary

This pipeline ensures:

- Internal BookStack remains precise and complete
- Public GitHub content is safe by construction
- Redaction is automated, consistent, and verifiable
- Documentation can be shared confidently

The separation between **authoring**, **sanitization**, and **publishing** is intentional and enforced.
