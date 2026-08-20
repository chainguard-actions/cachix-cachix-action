<!-- markdownlint-disable -->

# Hardening Report: cachix--cachix-action/v17

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **cachix--cachix-action/v17** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files use mutable tag-based refs instead of pinned 40-character SHA commits, making the workflows vulnerable to supply-chain attacks if the referenced action tags are moved. Failing references include: actions/checkout@v6, cachix/install-nix-action@v31, stefanzweifel/git-auto-commit-action@v7.

Locations:

- `.github/workflows/build.yml:13`
- `.github/workflows/build.yml:14`
- `.github/workflows/test.yml:27`
- `.github/workflows/test.yml:28`
- `.github/workflows/test.yml:43`
- `.github/workflows/test.yml:44`
- `.github/workflows/test.yml:62`
- `.github/workflows/test.yml:63`
- `.github/workflows/test.yml:80`
- `.github/workflows/test.yml:81`
- `.github/workflows/test.yml:95`
- `.github/workflows/test.yml:96`
- `.github/workflows/test.yml:116`
- `.github/workflows/test.yml:128`
- `.github/workflows/update-references.yml:13`
- `.github/workflows/update-references.yml:20`

### script-injection (severity: high)

GitHub Actions expressions are interpolated directly inside run: shell command strings, violating rule (a). (1) update-references.yml: `run: sed -i "s|cachix-action@v[0-9]\+|cachix-action@${{ github.ref_name }}|g" README.md` — ${{ github.ref_name }} is a tag name that could contain shell metacharacters and is injected directly into the shell command. (2) test.yml (nix-master job): `curl --location https://github.com/cachix/nar-toolbox/releases/download/v0.1.0/nar-toolbox-${{ matrix.system }} -O` and subsequent lines — ${{ matrix.system }} is interpolated directly into the run: block.

Locations:

- `.github/workflows/update-references.yml:17`
- `.github/workflows/test.yml:122`

### missing-permissions (severity: medium)

build.yml and test.yml have no top-level `permissions:` key and no job-level `permissions:` keys on any of their jobs. Without explicit permissions, workflows run with the default (potentially broad) token permissions. Each file should declare minimal required permissions at the top level or per job.

Locations:

- `.github/workflows/build.yml:1`
- `.github/workflows/test.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, missing-permissions

**Notes:**

Fixed all three findings across the three workflow files:

1. **unpinned-uses**: Pinned all mutable tag references to full 40-character SHAs with tag comments preserved: actions/checkout@v6 → SHA d23441a..., cachix/install-nix-action@v31 → SHA 13d8dd5..., stefanzweifel/git-auto-commit-action@v7 → SHA 4a55954...

2. **script-injection**: (a) In update-references.yml, moved `${{ github.ref_name }}` into an `env:` block as `REF_NAME` and referenced it as `${REF_NAME}` in the sed command. (b) In test.yml nix-master job, moved `${{ matrix.system }}` into an `env:` block as `MATRIX_SYSTEM` and referenced it as `${MATRIX_SYSTEM}` in the curl/chmod/exec commands.

3. **missing-permissions**: Added `permissions: {}` top-level block to build.yml and test.yml. update-references.yml already had appropriate `permissions: contents: write`.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. test.yml (nix-master job): The ${{ matrix.system }} value was already correctly isolated in the env: block as MATRIX_SYSTEM, and all three shell expansions (curl URL, chmod argument, command execution) were already inside double-quoted strings. No changes were needed.

2. update-references.yml (update-references job): Fixed the sed command that used ${REF_NAME} (from ${{ github.ref_name }}) as a sed replacement string. Converted to a multi-line run block that sanitizes the value first: uses printf '%s' to avoid flag injection, tr -d '\n\r' to strip newlines, and sed 's/[\\&|]/\\&/g' to escape sed delimiter characters (\, &, |). The sanitized safe_ref variable is then used in the sed replacement, preventing a malicious tag name from breaking out of the sed expression.

