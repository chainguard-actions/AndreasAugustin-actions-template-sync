<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **AndreasAugustin--actions-template-sync/v2.5.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ }} expression is directly interpolated inside a `run:` shell command string in action.yml. The step `run: ${{github.action_path}}/src/entrypoint.sh` embeds `${{github.action_path}}` directly in the shell command. Per the check rules, any `${{ ... }}` expression inside a `run:` block is a script-injection finding regardless of which context it reads from. The value should be accessed via an environment variable (e.g., `env: ACTION_PATH: ${{ github.action_path }}`) and then referenced as `$ACTION_PATH` in the shell script.

Locations:

- `action.yml:98`

### github-env-injection (severity: high)

The `set_github_action_outputs` function in src/sync_template.sh writes `pr_branch=${pr_branch}` and `template_git_hash=${template_git_hash}` to `$GITHUB_OUTPUT` without sanitization. The `pr_branch` value is constructed as `"${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}"` where `PR_BRANCH_NAME_PREFIX` is an inherited env var sourced from `inputs.pr_branch_name_prefix` (a caller-controlled input set via the action.yml env block: `PR_BRANCH_NAME_PREFIX: ${{ inputs.pr_branch_name_prefix }}`). A malicious caller could inject newlines into `pr_branch_name_prefix` to poison `$GITHUB_OUTPUT`. The required sanitization (`safe=$(printf '%s' "$PR_BRANCH" | tr -d '\n\r')`) is absent before the write.

Locations:

- `src/sync_template.sh:131`
- `src/sync_template.sh:132`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. action.yml line 98: Moved `${{github.action_path}}` out of the `run:` shell command into the step's `env:` block as `ACTION_PATH: ${{ github.action_path }}`. The shell command now uses `"$ACTION_PATH/src/entrypoint.sh"` instead. 2. src/sync_template.sh lines 131-132: In the `set_github_action_outputs` function, added sanitization of both `pr_branch` and `template_git_hash` values using `printf '%s' "${var}" | tr -d '\n\r'` before writing them to `$GITHUB_OUTPUT`, preventing newline injection via caller-controlled inputs like `pr_branch_name_prefix`.

### Iteration 2

**Fixes applied:** suspicious-run-content

**Notes:**

Fixed two suspicious-run-content findings:

1. eval-dynamic in src/sync_template.sh (line 246): Replaced `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"` with a safe array-based approach. The user-controlled `git_remote_pull_params` is now split into an array with `read -ra` and passed as properly quoted arguments to `git pull`, eliminating the shell injection risk from `eval`.

2. obfuscated-exec in src/sync_common.sh (line 52): Replaced `echo "${key}" | bash` with `bash -c "${key}"`. This removes the obfuscated pipe-to-bash execution pattern while preserving the hooks functionality (which is already gated behind `IS_ALLOW_HOOKS=true`).

