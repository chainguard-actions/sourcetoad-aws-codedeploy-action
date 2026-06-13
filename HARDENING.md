<!-- markdownlint-disable -->

# Hardening Report: sourcetoad--aws-codedeploy-action/v1.13.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **sourcetoad--aws-codedeploy-action/v1.13.0** was hardened automatically. 1 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

In deploy.sh, the variable ZIP_FILENAME is written to $GITHUB_OUTPUT without sanitization. When the user provides the `archive` input, ZIP_FILENAME is set directly to $INPUT_ARCHIVE (a caller-controlled value) and then written with `echo "zip_filename=$ZIP_FILENAME" >> "$GITHUB_OUTPUT"`. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, allowing a newline-injection attack that could poison the GitHub output environment. The same unsanitized pattern applies to the `etag=` write (though that value comes from AWS, it is still written without sanitization). Fix: apply `safe=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` before each write to $GITHUB_OUTPUT.

Locations:

- `deploy.sh:185`
- `deploy.sh:193`
- `deploy.sh:228`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection

**Notes:**

Fixed all three $GITHUB_OUTPUT writes in deploy.sh to apply newline sanitization before writing:
1. Line ~185: Added `safe_zip_filename=$(printf '%s' "$ZIP_FILENAME" | tr -d '\n\r')` before writing `zip_filename=` to $GITHUB_OUTPUT. ZIP_FILENAME can be set directly from user-controlled $INPUT_ARCHIVE, making this the highest-risk injection point.
2. Line ~193: Added `safe_zip_etag=$(printf '%s' "$ZIP_ETAG" | tr -d '\n\r')` before writing `etag=` to $GITHUB_OUTPUT. ZIP_ETAG comes from AWS but is still sanitized as a defense-in-depth measure.
3. Line ~228: Added `safe_deployment_id=$(printf '%s' "$DEPLOYMENT_ID" | tr -d '\n\r')` before writing `deployment_id=` to $GITHUB_OUTPUT. DEPLOYMENT_ID comes from AWS but is sanitized for consistency and defense-in-depth.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in deploy.sh:
1. Line 149: Replaced unquoted `$INPUT_CUSTOM_ZIP_FLAGS` in the zip command with a safe array expansion using `IFS=' ' read -r -a CUSTOM_ZIP_FLAGS_ARRAY <<< "$INPUT_CUSTOM_ZIP_FLAGS"` and `zip "${CUSTOM_ZIP_FLAGS_ARRAY[@]}"`. This prevents word-splitting and glob expansion of attacker-controlled content while preserving the ability to pass multiple flags.
2. Line 175: Replaced `if $INPUT_CODEDEPLOY_REGISTER_ONLY; then` (variable used as a command) with `if [ "$INPUT_CODEDEPLOY_REGISTER_ONLY" = "true" ]; then` (safe string comparison).
Also fixed the same command-execution pattern for `$INPUT_DRY_RUN` (changed from `if "$INPUT_DRY_RUN"; then` to `if [ "$INPUT_DRY_RUN" = "true" ]; then`) as it had the same vulnerability pattern.

