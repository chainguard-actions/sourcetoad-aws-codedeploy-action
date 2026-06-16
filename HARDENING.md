<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.14.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **sourcetoad--aws-codedeploy-action/v1.14.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In deploy.sh, when the `archive` input is provided, `ZIP_FILENAME` is set directly from `$INPUT_ARCHIVE` (a caller-controlled value) and then written to `$GITHUB_OUTPUT` without sanitization: `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"`. An attacker-controlled value containing newline characters could inject additional key=value pairs into GITHUB_OUTPUT, potentially overwriting other outputs. The required sanitization step (`printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'`) is absent before the write.

Locations:

- `deploy.sh:148`

### script-injection (severity: high)

Rule (b): `$INPUT_CUSTOM_ZIP_FLAGS` holds a caller-controlled input value and is intentionally expanded **unquoted** in the shell command `zip $INPUT_CUSTOM_ZIP_FLAGS -r --quiet "$ZIP_FILENAME" . -x "@$EXCLUSION_FILE"` (the `# shellcheck disable=SC2086` comment confirms this is deliberate). Unquoted expansion allows the shell to perform word-splitting and glob expansion on the attacker-controlled value, enabling injection of arbitrary `zip` flags or shell metacharacters (e.g. `;`, `|`, `$(...)`) that could execute unintended commands.

Locations:

- `deploy.sh:126`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed deploy.sh: (1) script-injection at line 126 — replaced unquoted `$INPUT_CUSTOM_ZIP_FLAGS` expansion (which allowed shell metacharacter injection) with a safe array: `IFS=' ' read -ra custom_zip_flags <<< "$INPUT_CUSTOM_ZIP_FLAGS"` then `zip "${custom_zip_flags[@]}"`, preserving multi-flag support without enabling word-splitting attacks; (2) github-env-injection at line 148 — added `safe_zip_filename=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` before writing to $GITHUB_OUTPUT to strip attacker-controlled newlines from the caller-supplied archive filename, preventing injection of additional output key=value pairs.

