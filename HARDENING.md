<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **AndreasAugustin--actions-template-sync/v2.4.2** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `run:` block in action.yml directly interpolates `${{github.action_path}}` inside the shell command string: `run: ${{github.action_path}}/src/entrypoint.sh`. Any `${{ ... }}` expression interpolated directly in a `run:` block is a script-injection risk because the value is substituted by the YAML template engine before the shell ever sees it, bypassing shell quoting.

Locations:

- `action.yml:88`

### github-env-injection (severity: high)

In `src/sync_template.sh`, the function `set_github_action_outputs` writes `echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"` without sanitization. The variable `pr_branch` is constructed as `PR_BRANCH_NAME_PREFIX_${TEMPLATE_GIT_HASH}`, where `PR_BRANCH_NAME_PREFIX` is an inherited process env var set directly from `inputs.pr_branch_name_prefix` (an untrusted caller-controlled input) in action.yml's `env:` block. No `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, allowing a newline-injection attack to poison subsequent GITHUB_OUTPUT entries.

Locations:

- `src/sync_template.sh:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

1. Fixed script-injection in action.yml: moved `${{github.action_path}}` out of the `run:` shell string into the step's `env:` block as `ACTION_PATH: ${{ github.action_path }}`, then referenced it as `$ACTION_PATH` in the run command. 2. Fixed github-env-injection in src/sync_template.sh: in the `set_github_action_outputs` function, both `pr_branch` and `template_git_hash` are now sanitized with `printf '%s' "${var}" | tr -d '\n\r'` before being written to `$GITHUB_OUTPUT`, preventing newline-injection attacks.

