<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.16.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sourcetoad--aws-codedeploy-action/v1.16.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

The workflow file .github/workflows/linting.yml references two Actions using mutable tag/branch refs instead of pinned 40-character commit SHAs. `actions/checkout@v4` uses a version tag and `azohra/shell-linter@latest` uses a branch name. Either could be silently updated to point to malicious code. They should be pinned to full SHA digests, e.g. `actions/checkout@<40-char-sha> # v4`.

Locations:

- `.github/workflows/linting.yml:11`
- `.github/workflows/linting.yml:14`

### missing-permissions (severity: medium)

The workflow file .github/workflows/linting.yml has no top-level `permissions:` key and the only job (`bash-lint`) also has no job-level `permissions:` key. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad (e.g. write access to contents). A minimal `permissions:` block (e.g. `contents: read`) should be added.

Locations:

- `.github/workflows/linting.yml:1`

### github-env-injection (severity: high)

In deploy.sh, the value of `$ZIP_FILENAME` is written directly to `$GITHUB_OUTPUT` without sanitization. When the `archive` input is provided by the caller, `ZIP_FILENAME` is set to `$INPUT_ARCHIVE` (a caller-controlled env var inherited from the calling workflow), meaning an attacker could inject newline characters to smuggle additional key=value pairs into `$GITHUB_OUTPUT`. The required sanitization step (`printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'`) is absent before the write. Similarly, `$ZIP_ETAG` and `$DEPLOYMENT_ID` are written to `$GITHUB_OUTPUT` without sanitization. The line `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"` is the primary violation since `ZIP_FILENAME` can directly reflect the user-supplied `archive` input.

Locations:

- `deploy.sh:175`
- `deploy.sh:191`
- `deploy.sh:210`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, github-env-injection

**Notes:**

1. unpinned-uses: Pinned actions/checkout@v4 to SHA 34e114876b0b11c390a56381ad16ebd13914f8d5 and azohra/shell-linter@latest to SHA 6bbeaa868df09c34ddc008e6030cfe89c03394a1 in .github/workflows/linting.yml, preserving original tags as comments.
2. missing-permissions: Added top-level `permissions: contents: read` block to .github/workflows/linting.yml.
3. github-env-injection: Sanitized all three GITHUB_OUTPUT writes in deploy.sh by introducing safe_* variables using `printf '%s' "$VAR" | tr -d '\n\r'` before writing ZIP_FILENAME, ZIP_ETAG, and DEPLOYMENT_ID to $GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in deploy.sh at line 148. Replaced the unquoted `$INPUT_CUSTOM_ZIP_FLAGS` expansion (with `# shellcheck disable=SC2086` suppressing the linter warning) with a bash array approach. The fix uses `IFS=' ' read -r -a custom_zip_flags_array <<< "$INPUT_CUSTOM_ZIP_FLAGS"` to safely split the space-delimited flags into an array, then expands with `"${custom_zip_flags_array[@]}"` to pass each flag as a separate properly-quoted argument to `zip`. This prevents shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) from being interpreted as shell commands while preserving the intended functionality of passing custom zip flags.

