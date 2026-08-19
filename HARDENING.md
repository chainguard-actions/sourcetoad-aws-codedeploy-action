<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.14.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sourcetoad--aws-codedeploy-action/v1.14.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In deploy.sh, the variable ZIP_FILENAME is written to $GITHUB_OUTPUT without sanitization. When the `archive` input is provided, ZIP_FILENAME is set directly from $INPUT_ARCHIVE (a workflow-caller-controlled env var): `ZIP_FILENAME="$INPUT_ARCHIVE"`. This value is then written unsanitized via `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"`. An attacker-controlled value containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT. The required sanitization step (`printf '%s' "$INPUT_ARCHIVE" | tr -d '\n\r'`) is absent before the write.

Locations:

- `deploy.sh:168`
- `deploy.sh:197`

### unpinned-uses (severity: high)

The linting workflow references two actions using mutable tag/branch refs instead of full 40-character commit SHAs, making the workflow vulnerable to supply-chain attacks if those tags are moved or compromised:
- `actions/checkout@v4` (tag ref)
- `azohra/shell-linter@latest` (branch ref — especially dangerous as `latest` always tracks the newest commit)

Locations:

- `.github/workflows/linting.yml:11`
- `.github/workflows/linting.yml:14`

### missing-permissions (severity: medium)

The linting workflow (.github/workflows/linting.yml) has no top-level `permissions:` key and no job-level `permissions:` key on the `bash-lint` job. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g., write access to contents). A minimal permissions block such as `permissions: read-all` or specific scopes (e.g., `contents: read`) should be added.

Locations:

- `.github/workflows/linting.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unpinned-uses, missing-permissions

**Notes:**

1. deploy.sh (line 168): Added sanitization of ZIP_FILENAME before writing to GITHUB_OUTPUT using `safe_zip_filename=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` and writing `$safe_zip_filename` instead. This prevents newline injection from attacker-controlled $INPUT_ARCHIVE values.
2. .github/workflows/linting.yml: Pinned `actions/checkout@v4` to full SHA `11d5960a326750d5838078e36cf38b85af677262` and `azohra/shell-linter@latest` to full SHA `6bbeaa868df09c34ddc008e6030cfe89c03394a1`, with tag comments preserved for readability.
3. .github/workflows/linting.yml: Added top-level `permissions: contents: read` block to restrict the workflow token to the minimum required permissions.

