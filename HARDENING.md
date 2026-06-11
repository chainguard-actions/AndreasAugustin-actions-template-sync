<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **AndreasAugustin--actions-template-sync/v2.5.1** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A `${{ ... }}` expression is directly interpolated inside a `run:` shell command string. The step `run: ${{github.action_path}}/src/entrypoint.sh` embeds `${{github.action_path}}` directly in the run block. Any expression interpolated directly into a run block — regardless of context — is a script-injection finding. The safe pattern is to use the pre-set environment variable `$GITHUB_ACTION_PATH` instead.

Locations:

- `action.yml:100`

### github-env-injection (severity: high)

The `set_github_action_outputs` function writes `echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"` where `pr_branch` is constructed as `"${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}"`. `PR_BRANCH_NAME_PREFIX` is sourced from `inputs.pr_branch_name_prefix` (user-controlled, set via the `env:` block in action.yml as `PR_BRANCH_NAME_PREFIX: ${{ inputs.pr_branch_name_prefix }}`). This value is written to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' "$VAR" | tr -d '\n\r'`), allowing a newline injection attack to set arbitrary output variables or environment variables in downstream steps.

Locations:

- `src/sync_template.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. Fixed script-injection in action.yml line 100: replaced `${{github.action_path}}/src/entrypoint.sh` with `$GITHUB_ACTION_PATH/src/entrypoint.sh` to use the pre-set environment variable instead of directly interpolating a GitHub Actions expression in the run block. 2. Fixed github-env-injection in src/sync_template.sh in the `set_github_action_outputs` function: added `safe_pr_branch=$(printf '%s' "${pr_branch}" | tr -d '\n\r')` and `safe_template_git_hash=$(printf '%s' "${template_git_hash}" | tr -d '\n\r')` before writing to $GITHUB_OUTPUT, preventing newline injection attacks from the user-controlled `PR_BRANCH_NAME_PREFIX` input.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in `src/sync_template.sh` at the `pull_source_changes` function (line 238). Replaced `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"` with a safe equivalent: (1) `read -ra pull_params_arr <<< "${git_remote_pull_params}"` to split the params string into an array without shell interpretation, and (2) `git pull "$source_repo" --tags "${pull_params_arr[@]}"` to invoke git directly with properly quoted arguments. This eliminates the `eval`-based injection vector while preserving the original multi-flag functionality.

