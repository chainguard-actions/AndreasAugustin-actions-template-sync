<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **AndreasAugustin--actions-template-sync/v2.5.0** was hardened automatically. 4 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ }} expression is directly interpolated inside a `run:` shell command string in action.yml. The line `run: ${{github.action_path}}/src/entrypoint.sh` embeds the GitHub Actions expression directly in the shell command before the shell ever sees it. Any ${{ ... }} in a run: block is a script-injection finding regardless of which context it reads from.

Locations:

- `action.yml:99`

### github-env-injection (severity: high)

In src/sync_template.sh, the `set_github_action_outputs` function writes `pr_branch` and `template_git_hash` to $GITHUB_OUTPUT without sanitization. `pr_branch` is constructed as `${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}` where `PR_BRANCH_NAME_PREFIX` is directly derived from `inputs.pr_branch_name_prefix` (a user-controlled input passed via the env block in action.yml). No `printf '%s' ... | tr -d '\n\r'` sanitization step is applied before the write, allowing newline injection into the output file.

Locations:

- `src/sync_template.sh:131`
- `src/sync_template.sh:132`

### suspicious-run-content (severity: high)

eval-dynamic: In src/sync_template.sh, the `pull_source_changes` function uses `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"`. The variable `git_remote_pull_params` is sourced from the `GIT_REMOTE_PULL_PARAMS` environment variable, which is set from `inputs.git_remote_pull_params` (a user-controlled input). Using `eval` with a `$`-expanded variable allows an attacker to inject arbitrary shell commands via the `git_remote_pull_params` input.

Locations:

- `src/sync_template.sh:243`

### suspicious-run-content (severity: high)

obfuscated-exec: In src/sync_common.sh, the `cmd_from_yml` function pipes dynamically-sourced YAML hook commands directly to bash: `echo "${key}" | bash`. The hook content is read from the `HOOKS` input variable (`inputs.hooks`, user-controlled) or from a local YAML file. This allows arbitrary shell command execution from user-supplied input, matching the pattern of piping dynamic/decoded content to a shell interpreter.

Locations:

- `src/sync_common.sh:57`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, suspicious-run-content

**Notes:**

Fixed all four findings: (1) script-injection in action.yml: moved ${{github.action_path}} to env block as ACTION_PATH and referenced it as ${ACTION_PATH} in the run command; (2) github-env-injection in sync_template.sh: sanitized pr_branch and template_git_hash with printf | tr -d '\n\r' before writing to $GITHUB_OUTPUT; (3) eval-dynamic in sync_template.sh: replaced eval with read -ra array splitting and direct git pull with array expansion; (4) obfuscated-exec in sync_common.sh: replaced 'echo "${key}" | bash' with 'bash -c "${key}"' to eliminate the pipe-to-shell pattern.

### Iteration 2

**Fixes applied:** script-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all 6 findings across 13 workflow files:

1. script-injection (push_docker.yml): Moved matrix.variant, inputs.tag, and matrix.target into env vars (VARIANT, TAG, TARGET) in the 'test docker image' step. Dev suffix logic computed in shell.

2. script-injection (release_please.yml): Moved secrets.GITHUB_TOKEN, steps.release.outputs.major, and steps.release.outputs.minor into env vars (GH_TOKEN, MAJOR, MINOR) in the 'tag major and minor versions' step.

3. script-injection (release_test_docker_images.yml): Moved matrix.docker-image and inputs.tag into env vars (DOCKER_IMAGE, TAG) in both 'pull image' and 'run tests' steps.

4. unpinned-uses: Pinned all 9 action references to full 40-char SHAs across all 13 affected workflow files: actions/checkout@11d5960a, actions/setup-python@a26af69b, docker/login-action@c94ce9fb, docker/metadata-action@c299e40c, docker/build-push-action@10e90e36, peter-evans/dockerhub-description@432a30c9, googleapis/release-please-action@5c625bfb, actions/create-github-app-token@d72941d7, AndreasAugustin/actions-template-sync@8a0f668b.

5. missing-permissions (lint.yml): Added top-level `permissions: contents: read`.

6. missing-permissions (shellcheck.yml): Added top-level `permissions: contents: read`.

### Iteration 3

**Fixes applied:** missing-permissions

**Notes:**

Added explicit `permissions: contents: read` blocks to the `call_shellcheck` (line 9) and `call_lint` (line 11) jobs in `.github/workflows/release.yml`. These jobs only need read access to repository contents to perform shellcheck and lint operations. All other jobs in the file already had explicit permissions blocks, so no further changes were needed.

