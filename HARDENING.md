<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **AndreasAugustin--actions-template-sync/v2.5.2** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `run:` block in action.yml directly interpolates the GitHub Actions expression `${{github.action_path}}` inside the shell command string: `run: ${{github.action_path}}/src/entrypoint.sh`. Any `${{ ... }}` expression interpolated directly into a `run:` script is a script-injection risk because the value flows through YAML template substitution before the shell ever sees it. The safe alternative is to use the pre-set environment variable `$GITHUB_ACTION_PATH` instead.

Locations:

- `action.yml:101`

### github-env-injection (severity: high)

The `set_github_action_outputs` function in src/sync_template.sh writes `pr_branch=${pr_branch}` to `$GITHUB_OUTPUT` without sanitization. The value of `pr_branch` is composed of `PR_BRANCH_NAME_PREFIX` (inherited from the calling workflow via the `inputs.pr_branch_name_prefix` input, set as env var `PR_BRANCH_NAME_PREFIX` in action.yml) concatenated with a git short hash. Because `PR_BRANCH_NAME_PREFIX` is user-controlled and no `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, an attacker can inject newlines into `$GITHUB_OUTPUT` to set arbitrary output variables or environment variables in downstream steps.

Locations:

- `src/sync_template.sh:130`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. action.yml line 101: Replaced `${{github.action_path}}/src/entrypoint.sh` with `$GITHUB_ACTION_PATH/src/entrypoint.sh` to eliminate the script-injection risk from YAML template substitution. 2. src/sync_template.sh set_github_action_outputs function: Added newline sanitization using `printf '%s' ... | tr -d '\n\r'` for both `pr_branch` and `template_git_hash` before writing to `$GITHUB_OUTPUT`, preventing github-env-injection via user-controlled `PR_BRANCH_NAME_PREFIX` input.

