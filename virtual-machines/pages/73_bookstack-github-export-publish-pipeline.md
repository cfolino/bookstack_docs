This page documents the **automated but explicitly staged** pipeline used to export internal BookStack documentation, sanitize sensitive data, and publish a public-safe mirror to GitHub.

The internal BookStack instance remains the authoritative source of truth.
GitHub is treated as a **read-only mirror** for portfolio and reference purposes.

---

## Pipeline Overview

The pipeline is intentionally split into **clear, single-responsibility stages** that must be run in order:

1. Export – Lossless extraction from BookStack
2. Sanitize – Deterministic redaction of sensitive data
3. Publish – Version-controlled push to GitHub
4. Verify – Continuous integration leak scanning in GitHub

The separation between stages is intentional to ensure safety, auditability, and operator awareness.

---

## Directory Layout

/home/bookstack/
├── bookstack-export.sh
├── sanitize-export.sh
├── publish-bookstack-export.sh
├── bookstack_export_YYYYMMDD_HHMMSS/
│   └── (raw BookStack export, immutable)
└── bookstack_export_sanitized/
    └── (sanitized GitHub mirror, version-controlled)

- Timestamped export directories are immutable and never modified
- bookstack_export_sanitized is the only directory under Git version control
- Raw exports are never pushed to GitHub

---

## Step 1: Export (Lossless)

Script: bookstack-export.sh

This script:
- Uses the BookStack API
- Exports books, chapters, and pages
- Preserves Markdown exactly as authored
- Writes output to a timestamped directory

No sanitization or redaction occurs at this stage.

Command:

./bookstack-export.sh

---

## Step 2: Sanitize (Redaction)

Script: sanitize-export.sh

This script performs deterministic redaction of sensitive internal data inside the sanitized export directory.

Redacted content:
- Internal domains
- Internal email addresses
- RFC1918 IP addresses
- Disk UUIDs and UUID-based mount paths

Preserved content:
- Directory structure
- Script logic and commands
- Public keys
- Standard Linux paths

Example redactions:

192.168.x.x              → 192.168.x.x
<internal-host>            → <internal-domain>
<redacted-email>         → <redacted-email>
dev-disk-by-uuid-<uuid>    → dev-disk-by-uuid-<uuid-redacted>

Command:

./sanitize-export.sh ~/bookstack_export_sanitized

---

## Step 3: Publish (Version-Controlled Push)

Script: publish-bookstack-export.sh

This script is the **only** component allowed to push content to GitHub.

It performs the following:
1. Synchronizes the latest export into the sanitized directory
2. Stages sanitized content for Git
3. Commits changes using a timestamped message
4. Pushes to the GitHub mirror branch

No force-pushes are performed.

Command:

./publish-bookstack-export.sh

---

## Step 4: CI Leak Scanning (GitHub)

A GitHub Actions workflow runs on every push and pull request to enforce safety guarantees.

The CI scan validates:
- No private key material outside documentation
- No internal domains outside documentation
- No internal email addresses outside documentation
- No raw RFC1918 IP addresses
- No cloud API keys or bearer tokens

This provides a second, external safety boundary beyond local sanitization.

---

## Repository README Handling

README.md is repository-owned metadata and is excluded from export synchronization.
It explains that the repository is a public mirror and persists across exports.

---

## Normal Usage Workflow

./bookstack-export.sh
./sanitize-export.sh ~/bookstack_export_sanitized
./publish-bookstack-export.sh

---

## Summary

This pipeline ensures:
- Internal documentation remains complete and authoritative
- Public documentation is safe by construction
- Redaction is deterministic and repeatable
- Git history is preserved without rewriting
- CI enforces an additional independent safety boundary

The explicit separation between authoring, sanitization, publishing, and verification is intentional.
