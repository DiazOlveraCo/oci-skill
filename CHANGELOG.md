# Changelog — oci

All notable changes to this skill are recorded here. Newest first.

## 2026-11-12 — references: add generic download/install workflow page
- Added `references/download-install-workflow.md`: a dedicated page covering only the general workflow for downloading files with `curl` and installing them locally on a Linux VM.
- Contains no package names, versions, or vendor URLs — artifact names are resolved at runtime.

## 2026-11-12 — references: revert oci-cli-install-rpm.md edit
- Reverted the 19:15 edit that added verbose curl flags and rewrote the core rule toward `dnf localinstall`, leaving the file internally inconsistent (one edit applied, follow-up aborted).
- Restored pre-edit content. The reverted edit had conflated general RPM download guidance with OCI CLI installation.

## 2026-11-12 — baseline
- Changelog initialized; existing skill content committed as-is.
