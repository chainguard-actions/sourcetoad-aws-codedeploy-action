<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.15.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sourcetoad--aws-codedeploy-action/v1.15.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

The workflow file linting.yml references two actions using mutable tag/branch refs instead of full 40-character commit SHAs:
- `uses: actions/checkout@v4` (tag ref)
- `uses: azohra/shell-linter@latest` (branch ref — especially dangerous as `latest` always tracks HEAD)

These should be pinned to immutable SHA digests, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `.github/workflows/linting.yml:11`
- `.github/workflows/linting.yml:14`

### missing-permissions (severity: medium)

The workflow file linting.yml has no top-level `permissions:` key and the only job (`bash-lint`) also has no job-level `permissions:` key. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad. A minimal permissions block (e.g. `permissions: read-all` or specific scopes) should be added.

Locations:

- `.github/workflows/linting.yml:1`

### github-env-injection (severity: high)

In deploy.sh, the variable `ZIP_FILENAME` is assigned directly from the user-controlled input `$INPUT_ARCHIVE` (corresponding to `inputs.archive` in action.yaml) in the else branch:

    ZIP_FILENAME="$INPUT_ARCHIVE"

This value is then written to `$GITHUB_OUTPUT` without any newline-stripping sanitization:

    echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"

An attacker-controlled value containing newlines could inject arbitrary key=value pairs into `$GITHUB_OUTPUT`, potentially overwriting other outputs or injecting environment variables. The required sanitization step (`printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'`) is missing before the write.

Locations:

- `deploy.sh:168`

### script-injection (severity: high)

Rule (b) violation: In deploy.sh, the user-controlled env var `$INPUT_CUSTOM_ZIP_FLAGS` (sourced from `inputs.custom_zip_flags`) is expanded **unquoted** inside the `zip` command:

    zip $INPUT_CUSTOM_ZIP_FLAGS -r --quiet "$ZIP_FILENAME" . -x "@$EXCLUSION_FILE"

The `# shellcheck disable=SC2086` comment confirms the author intentionally suppressed the quoting warning, but an unquoted expansion of a workflow-controllable variable allows shell word-splitting and glob expansion of attacker-supplied content, enabling command injection via metacharacters. The variable should be double-quoted or handled via an array.

Locations:

- `deploy.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, github-env-injection, script-injection

**Notes:**

Fixed all four findings:
1. **unpinned-uses** (linting.yml lines 11, 14): Pinned `actions/checkout@v4` to SHA `34e114876b0b11c390a56381ad16ebd13914f8d5` and `azohra/shell-linter@latest` to SHA `6bbeaa868df09c34ddc008e6030cfe89c03394a1`, with original tag/branch preserved as comments.
2. **missing-permissions** (linting.yml line 1): Added `permissions: {}` top-level block — no permissions are needed for this linting workflow.
3. **github-env-injection** (deploy.sh line 168): Added sanitization of `ZIP_FILENAME` before writing to `$GITHUB_OUTPUT` using `safe_zip_filename=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` and writing `safe_zip_filename` instead.
4. **script-injection** (deploy.sh line 130): Replaced the unquoted `$INPUT_CUSTOM_ZIP_FLAGS` expansion (with `# shellcheck disable=SC2086`) with a bash array approach: `read -ra custom_zip_flags <<< "$INPUT_CUSTOM_ZIP_FLAGS"` followed by `zip "${custom_zip_flags[@]}" ...`, which properly controls word-splitting without enabling shell injection.

