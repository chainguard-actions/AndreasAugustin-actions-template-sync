<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **AndreasAugustin--actions-template-sync/v2.6.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ }} expression is directly interpolated inside the run: shell command string in action.yml. The line `run: ${{github.action_path}}/src/entrypoint.sh` embeds `${{github.action_path}}` directly in the shell command. Per the check rules, any `${{ ... }}` expression inside a run: block is a script-injection finding regardless of which context it reads from. The safe alternative is to use the pre-set environment variable `$GITHUB_ACTION_PATH` instead.

Locations:

- `action.yml:114`

### github-env-injection (severity: high)

The set_github_action_outputs function in src/sync_template.sh writes `pr_branch=${pr_branch}` to $GITHUB_OUTPUT without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). The `pr_branch` value is constructed as `"${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}"` where `PR_BRANCH_NAME_PREFIX` is sourced directly from `inputs.pr_branch_name_prefix` (a caller-controlled input set via the env: block in action.yml as `PR_BRANCH_NAME_PREFIX: ${{ inputs.pr_branch_name_prefix }}`). A malicious caller could inject newlines into this input to poison $GITHUB_OUTPUT with arbitrary key-value pairs.

Locations:

- `src/sync_template.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. Fixed script-injection in action.yml: replaced `${{github.action_path}}/src/entrypoint.sh` with `"$GITHUB_ACTION_PATH/src/entrypoint.sh"` in the run: block, using the pre-set GITHUB_ACTION_PATH environment variable instead of a ${{ }} expression. 2. Fixed github-env-injection in src/sync_template.sh: added `printf '%s' ... | tr -d '\n\r'` sanitization for pr_branch, template_git_hash, and pr_number before writing them to $GITHUB_OUTPUT, preventing newline injection attacks from caller-controlled inputs like PR_BRANCH_NAME_PREFIX.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in three workflow files:

1. `.github/workflows/release_please.yml`: Added `env:` block with `GITHUB_TOKEN`, `RELEASE_MAJOR`, and `RELEASE_MINOR` variables to the 'tag major and minor versions' step. Replaced all `${{ secrets.GITHUB_TOKEN }}` and `${{ steps.release.outputs.major/minor }}` interpolations in the `run:` block with `${GITHUB_TOKEN}`, `${RELEASE_MAJOR}`, and `${RELEASE_MINOR}` shell variable references (double-quoted throughout).

2. `.github/workflows/push_docker.yml`: Added `env:` block with `MATRIX_VARIANT`, `INPUT_TAG`, and `MATRIX_TARGET` variables to the 'test docker image' step. Replaced all `${{ matrix.variant }}`, `${{ inputs.tag }}`, and the ternary `${{ matrix.target == 'dev' && '-dev' || '' }}` expressions with shell variable references. The ternary is now computed in shell using an `if` statement that sets `DEV_SUFFIX`.

3. `.github/workflows/release_test_docker_images.yml`: Added `env:` blocks with `DOCKER_IMAGE` and `INPUT_TAG` variables to both the 'pull image' and 'run tests' steps. Replaced all `${{ matrix.docker-image }}` and `${{ inputs.tag }}` interpolations with double-quoted shell variable references `"${DOCKER_IMAGE}:${INPUT_TAG}"`.

