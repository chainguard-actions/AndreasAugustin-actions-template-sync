<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **AndreasAugustin--actions-template-sync/v2.5.2** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): a GitHub Actions expression is placed directly in a run: shell command in the composite step 'github sync': `run: ${{github.action_path}}/src/entrypoint.sh`. The value is substituted into the script text before the shell runs it, so it is never quoted. A path containing spaces or shell metacharacters would be parsed by the shell. Fix: use the runner-provided env var instead, with quotes: `run: "$GITHUB_ACTION_PATH/src/entrypoint.sh"`.

Locations:

- `action.yml:101`

### github-env-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The `set_github_action_outputs` function in src/sync_template.sh writes `pr_branch=${pr_branch}` to `$GITHUB_OUTPUT` without sanitization. The value of `pr_branch` is composed of `PR_BRANCH_NAME_PREFIX` (inherited from the calling workflow via the `inputs.pr_branch_name_prefix` input, set as env var `PR_BRANCH_NAME_PREFIX` in action.yml) concatenated with a git short hash. Because `PR_BRANCH_NAME_PREFIX` is user-controlled and no `printf '%s' ... | tr -d '\n\r'` sanitization is applied before the write, an attacker can inject newlines into `$GITHUB_OUTPUT` to set arbitrary output variables or environment variables in downstream steps.

Locations:

- `src/sync_template.sh:130`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index a402947..9fa1799 100644
--- a/action.yml
+++ b/action.yml
@@ -97,8 +97,25 @@ runs:
   using: "composite"
   # image: "src/Dockerfile"
   steps:
     - name: github sync
-      run: ${{github.action_path}}/src/entrypoint.sh
+      run: $GITHUB_ACTION_PATH/src/entrypoint.sh
       # working-directory: src/
       shell: bash
       id: sync

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed src/sync_template.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/src/sync_template.sh b/src/sync_template.sh
index 908348b..f2fd336 100755
--- a/src/sync_template.sh
+++ b/src/sync_template.sh
@@ -139,8 +139,10 @@ function set_github_action_outputs() {
     info "env var 'GITHUB_RUN_ID' is empty -> no github action workflow"
   else
     # https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-output-parameter
-    echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"
-    echo "template_git_hash=${template_git_hash}" >> "$GITHUB_OUTPUT"
+    safe_pr_branch=$(printf '%s' "${pr_branch}" | tr -d '\n\r')
+    safe_template_git_hash=$(printf '%s' "${template_git_hash}" | tr -d '\n\r')
+    echo "pr_branch=${safe_pr_branch}" >> "$GITHUB_OUTPUT"
+    echo "template_git_hash=${safe_template_git_hash}" >> "$GITHUB_OUTPUT"
   fi
   echo "::endgroup::"
 }

```

Locations:

- `src/sync_template.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, prior-fix

**Notes:**

action.yml: replaced inline ${{github.action_path}} with quoted "$GITHUB_ACTION_PATH/src/entrypoint.sh" (bash shell). src/sync_template.sh: strip \n\r from pr_branch and template_git_hash before writing to $GITHUB_OUTPUT (re-applies the prior fix).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

src/sync_template.sh pull_source_changes no longer uses eval. It now runs git pull "$source_repo" --tags "${pull_params[@]}". Because upstream used eval, the shell honoured quotes inside git_remote_pull_params (e.g. -X theirs). To keep that, pull_params is split with the quote-aware xargs/sh -c printf method. Nothing gets evaluated, so ;, |, $() and backticks reach git as plain text. If the value has an unmatched quote, the script stops with an error before splitting, so arguments are never silently dropped. Upstream's eval would have hit a parse error and carried on with pull_has_issues=true instead. An empty value adds no arguments. The script already uses bash (#!/usr/bin/env bash), so arrays and process substitution are fine. Not tested by running it.

