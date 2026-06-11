<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **AndreasAugustin--actions-template-sync/v2.5.3** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The run: block in action.yml directly interpolates a ${{ }} expression (`${{github.action_path}}`) inside the shell command string. Per the script-injection check (rule a), any GitHub Actions expression interpolated directly inside a run: block is a script injection risk, as the value flows through YAML template substitution before the shell ever sees it. The offending line is: `run: ${{github.action_path}}/src/entrypoint.sh`. This should be replaced with the environment variable `$GITHUB_ACTION_PATH` instead.

Locations:

- `action.yml:100`

### github-env-injection (severity: high)

In src/sync_template.sh, the set_github_action_outputs() function writes `pr_branch=${pr_branch}` to $GITHUB_OUTPUT without sanitization. The `pr_branch` value is constructed as `${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}`, where `PR_BRANCH_NAME_PREFIX` is an inherited env var set from `inputs.pr_branch_name_prefix` (a caller-controlled input). Writing this unsanitized value to $GITHUB_OUTPUT allows a newline injection attack. The required sanitization step (`printf '%s' "$pr_branch" | tr -d '\n\r'`) must be applied before the write.

Locations:

- `src/sync_template.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. action.yml line 100: Replaced `${{github.action_path}}/src/entrypoint.sh` with `$GITHUB_ACTION_PATH/src/entrypoint.sh` to eliminate the script injection risk from direct YAML expression interpolation in a run: block. The built-in GITHUB_ACTION_PATH env var is the correct replacement. 2. src/sync_template.sh set_github_action_outputs(): Added sanitization for all three output values (pr_branch, template_git_hash, pr_number) using `printf '%s' "$VAR" | tr -d '\n\r'` before writing to $GITHUB_OUTPUT, preventing newline injection from caller-controlled inputs such as pr_branch_name_prefix.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the eval-based shell injection in pull_source_changes() in src/sync_template.sh. Replaced `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"` with a safe direct git invocation: split git_remote_pull_params into an array using `read -ra pull_params <<< "${git_remote_pull_params}"`, then call `git pull "${source_repo}" --tags "${pull_params[@]}"`. This properly quotes source_repo to prevent word splitting/glob expansion and passes git_remote_pull_params flags as individual array elements rather than through eval, eliminating the shell metacharacter injection risk.

