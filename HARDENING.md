<!-- markdownlint-disable -->

# Hardening Report: AndreasAugustin--actions-template-sync/v2.5.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **AndreasAugustin--actions-template-sync/v2.5.3** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): a GitHub Actions expression is substituted straight into the composite step's run: script: `run: ${{github.action_path}}/src/entrypoint.sh`. Pass it in through the environment and quote it instead, e.g. `run: "$GITHUB_ACTION_PATH/src/entrypoint.sh"`.

Locations:

- `action.yml:104`

### github-env-injection (severity: high)

set_github_action_outputs writes `echo "pr_branch=${pr_branch}" >> "$GITHUB_OUTPUT"` (plus template_git_hash and pr_number) without stripping newlines first. pr_branch is built from PR_BRANCH="${PR_BRANCH_NAME_PREFIX}_${TEMPLATE_GIT_HASH}", and PR_BRANCH_NAME_PREFIX comes from inputs.pr_branch_name_prefix, which the caller controls. A newline in that value can inject extra outputs. Run `printf '%s' ... | tr -d '\n\r'` on each value before writing it.

Locations:

- `src/sync_template.sh:144`

### script-injection (severity: high)

Sub-rule (b): the supporting script runs its input through eval. In pull_source_changes, `eval "git pull ${source_repo} --tags ${git_remote_pull_params}"` puts GIT_REMOTE_PULL_PARAMS (from inputs.git_remote_pull_params) and SOURCE_REPO into a string that eval runs as shell code, so shell metacharacters in those values become commands. Use an argument array instead, e.g. read -ra params <<< "$git_remote_pull_params"; git pull "$source_repo" --tags "${params[@]}".

Locations:

- `src/sync_template.sh:277`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index ccdb169..7c3f7ca 100644
--- a/action.yml
+++ b/action.yml
@@ -100,8 +100,25 @@ runs:
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
index 3941f40..56dd626 100755
--- a/src/sync_template.sh
+++ b/src/sync_template.sh
@@ -140,10 +140,14 @@ function set_github_action_outputs() {
     info "env var 'GITHUB_RUN_ID' is empty -> no github action workflow"
   else
     # https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#setting-an-output-parameter
+    # Sanitize values to prevent newline injection before writing to GITHUB_OUTPUT
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
   echo "::endgroup::"
 }
@@ -275,7 +279,8 @@ function pull_source_changes() {
   local source_repo=$1
   local git_remote_pull_params=$2
 
-  eval "git pull ${source_repo} --tags ${git_remote_pull_params}" || pull_has_issues=true
+  read -ra pull_params <<< "${git_remote_pull_params}"
+  git pull "${source_repo}" --tags "${pull_params[@]}" || pull_has_issues=true
 
   info "finished pulling from the source."
   info "logging out from source ${SOURCE_REPO_HOSTNAME}."

```

Locations:

- `src/sync_template.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, prior-fix

**Notes:**

action.yml: the run step now calls the script as "$GITHUB_ACTION_PATH/src/entrypoint.sh", quoted, instead of using the ${{github.action_path}} expression. src/sync_template.sh: before writing pr_branch, template_git_hash and pr_number to GITHUB_OUTPUT, newlines are now stripped with printf | tr -d '\n\r'. In pull_source_changes, the eval is gone. The script now runs git pull "$source_repo" --tags "${pull_params[@]}", and pull_params is filled using the quote-aware xargs split, so quotes in git_remote_pull_params work the same way they did under eval, but nothing in the value is executed. An unmatched quote now sets pull_has_issues, which is the same outcome the old eval syntax error led to.

