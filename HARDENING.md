<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **AndreasAugustin--actions-template-sync/v2.6.0** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): a GitHub Actions expression is placed directly in a run: block in action.yml: `run: ${{github.action_path}}/src/entrypoint.sh`. Use the shell variable instead: `run: "$GITHUB_ACTION_PATH/src/entrypoint.sh"`.

Locations:

- `action.yml:114`

### script-injection (severity: high)

Rule (b): workflow-controlled input is run through eval in a supporting script that the action's run: step calls. `eval "git pull ${source_repo} --tags ${git_remote_pull_params} ${source_pull_ref}"` and `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"` build the command from GIT_REMOTE_PULL_PARAMS (inputs.git_remote_pull_params), SOURCE_REPO and SOURCE_BRANCH or a tag-derived ref. That lets shell metacharacters in those values run arbitrary commands. Build an argument array and run git without eval.

Locations:

- `src/sync_template.sh:309`
- `src/sync_template.sh:311`

### github-env-injection (severity: high)

set_github_action_outputs writes `echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"` (and the template_git_hash and pr_number lines) without sanitizing the values. pr_branch comes from PR_BRANCH_NAME_PREFIX (inputs.pr_branch_name_prefix), which the caller controls. A newline in the input could add extra output entries. Run printf '%s' ... | tr -d '\n\r' on each value before writing it.

Locations:

- `src/sync_template.sh:167`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 5752201..e9ca141 100644
--- a/action.yml
+++ b/action.yml
@@ -110,8 +110,25 @@ runs:
   using: "composite"
   # image: "src/Dockerfile"
   steps:
     - name: github sync
-      run: ${{github.action_path}}/src/entrypoint.sh
+      run: "$GITHUB_ACTION_PATH/src/entrypoint.sh"
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
index 4a441c2..190e4dc 100755
--- a/src/sync_template.sh
+++ b/src/sync_template.sh
@@ -163,10 +163,14 @@ function set_github_action_outputs() {
     info "env var 'GITHUB_RUN_ID' is empty -> no github action workflow"
   else
     # https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-output-parameter
+    # Sanitize values to prevent newline injection into $GITHUB_OUTPUT
     # shellcheck disable=SC2129
-    echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"
-    echo "template_git_hash=${template_git_hash}" >> "$GITHUB_OUTPUT"
-    echo "pr_number=${pr_number}" >> "$GITHUB_OUTPUT"
+    safe_pr_branch=$(printf '%s' "${pr_branch}" | tr -d '\n\r')
+    safe_template_git_hash=$(printf '%s' "${template_git_hash}" | tr -d '\n\r')
+    safe_pr_number=$(printf '%s' "${pr_number}" | tr -d '\n\r')
+    echo "pr_branch=${safe_pr_branch}" >> "$GITHUB_OUTPUT"
+    echo "template_git_hash=${safe_template_git_hash}" >> "$GITHUB_OUTPUT"
+    echo "pr_number=${safe_pr_number}" >> "$GITHUB_OUTPUT"
   fi
   end_group
   return $?

```

Locations:

- `src/sync_template.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, prior-fix

**Notes:**

action.yml: the step now runs "$GITHUB_ACTION_PATH/src/entrypoint.sh" instead of the inline ${{github.action_path}} expression. src/sync_template.sh: removed the eval in pull_source_changes. git pull now gets source_repo and source_pull_ref as separate quoted arguments. git_remote_pull_params is split into an array the way eval used to split it, honoring quotes, using xargs with the guard and the -r, unmatched-quote and sh -c built-in printf safeguards. It is then passed as "${pull_params[@]}". set -u is off in the script, so an empty array expands safely. One behaviour change: if git_remote_pull_params has an unmatched quote, the script now stops under set -e. Before, the eval syntax error was swallowed by '|| pull_has_issues=true' and the run carried on. set_github_action_outputs now strips newlines from pr_branch, template_git_hash and pr_number before writing them to $GITHUB_OUTPUT. Both prior-fix patches are re-applied.

