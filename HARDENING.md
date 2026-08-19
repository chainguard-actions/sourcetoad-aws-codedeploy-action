<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.13.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sourcetoad--aws-codedeploy-action/v1.13.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

The workflow file .github/workflows/linting.yml references two actions using mutable tag/branch refs instead of full 40-character SHA digests, making the workflow vulnerable to supply-chain attacks if those tags are moved or compromised:
- `uses: actions/checkout@v4` (tag ref)
- `uses: azohra/shell-linter@latest` (branch ref — especially dangerous as 'latest' is a floating ref)

Locations:

- `.github/workflows/linting.yml:11`
- `.github/workflows/linting.yml:14`

### permissions (severity: medium)

The workflow file .github/workflows/linting.yml has no top-level `permissions:` key and no job-level `permissions:` key on any of its jobs. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g. write access to contents). A minimal permissions block such as `permissions: read-all` or specific scopes should be added.

Locations:

- `.github/workflows/linting.yml:1`

### github-env-injection (severity: high)

In deploy.sh, the variable ZIP_FILENAME is written to $GITHUB_OUTPUT without sanitization. When the `archive` input is provided, ZIP_FILENAME is set directly from the user-controlled input $INPUT_ARCHIVE (line: `ZIP_FILENAME="$INPUT_ARCHIVE"`), and then written unsanitized: `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"`. An attacker-controlled value containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT, potentially overwriting other outputs. The fix requires applying `printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'` before the write.

Locations:

- `deploy.sh:161`
- `deploy.sh:175`

### script-injection (severity: high)

Rule (b) violation: The user-controlled input $INPUT_CUSTOM_ZIP_FLAGS is expanded unquoted in the zip command in deploy.sh: `zip $INPUT_CUSTOM_ZIP_FLAGS -r --quiet "$ZIP_FILENAME" . -x "@$EXCLUSION_FILE"`. The shellcheck warning SC2086 is explicitly suppressed with a disable comment, confirming the intentional unquoted expansion. An attacker who controls the `custom_zip_flags` input can inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) to execute arbitrary commands. The value should be quoted or handled via an array.

Locations:

- `deploy.sh:158`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, permissions, github-env-injection, script-injection

**Notes:**

Fixed all four findings: (1) Pinned actions/checkout@v4 to SHA 34e114876b0b11c390a56381ad16ebd13914f8d5 and azohra/shell-linter@latest to SHA 6bbeaa868df09c34ddc008e6030cfe89c03394a1 in linting.yml. (2) Added `permissions: contents: read` top-level block to linting.yml. (3) Sanitized ZIP_FILENAME before writing to $GITHUB_OUTPUT using `printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'` to prevent newline injection. (4) Replaced unquoted `$INPUT_CUSTOM_ZIP_FLAGS` expansion (and its shellcheck disable comment) with a bash array (`read -ra zip_flags <<< "$INPUT_CUSTOM_ZIP_FLAGS"`) so custom flags are word-split safely without allowing shell metacharacter injection.

