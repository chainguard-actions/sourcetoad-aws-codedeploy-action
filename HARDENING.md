<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.15.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **sourcetoad--aws-codedeploy-action/v1.15.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In deploy.sh, the variable ZIP_FILENAME is assigned from $INPUT_ARCHIVE (a user-controlled action input, `archive`) and then written directly to $GITHUB_OUTPUT without sanitization: `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"`. An attacker-controlled value containing newlines could inject arbitrary key=value pairs into GITHUB_OUTPUT, potentially overwriting other outputs. The required sanitization step (`printf '%s' "$ZIP_FILENAME" | tr -d '\n\r'`) is absent.

Locations:

- `deploy.sh:155`
- `deploy.sh:175`

### script-injection (severity: high)

Sub-rule (b): In deploy.sh, the user-controlled input $INPUT_CUSTOM_ZIP_FLAGS (action input `custom_zip_flags`) is expanded **unquoted** in the shell command: `zip $INPUT_CUSTOM_ZIP_FLAGS -r --quiet "$ZIP_FILENAME" . -x "@$EXCLUSION_FILE"`. The script even includes `# shellcheck disable=SC2086` to suppress the linter warning about this. An attacker supplying shell metacharacters or extra flags via `custom_zip_flags` can cause word-splitting and arbitrary flag injection into the zip command.

Locations:

- `deploy.sh:148`
- `deploy.sh:149`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two security issues in deploy.sh: (1) script-injection: replaced unquoted `$INPUT_CUSTOM_ZIP_FLAGS` expansion (with shellcheck disable comment) with a safe bash array approach using `read -ra CUSTOM_ZIP_FLAGS_ARRAY <<< "$INPUT_CUSTOM_ZIP_FLAGS"` and `zip "${CUSTOM_ZIP_FLAGS_ARRAY[@]}" ...`, preventing word-splitting and flag injection; (2) github-env-injection: sanitized `ZIP_FILENAME` before writing to $GITHUB_OUTPUT by adding `safe_zip_filename=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` and using the sanitized variable in the echo statement, preventing newline injection into GITHUB_OUTPUT.

