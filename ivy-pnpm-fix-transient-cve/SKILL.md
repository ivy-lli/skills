---
name: ivy-pnpm-fix-transient-cve
description: 'Check ivy pnpm repositories for open CVEs in transient dependencies on master, release/12, and release/10 worktrees, apply safe audit fixes, validate results, and open PRs only when real dependency changes are introduced.'
argument-hint: 'Optional: run on all default ivy pnpm repos or a provided subset of absolute repo paths'
---

# Ivy PNPM Transient CVE Fix

## When To Use
- You need to scan ivy pnpm repositories for vulnerable transient dependencies that Renovate may not auto-upgrade.
- You need to scan ivy pnpm repositories on `master`, `release/12`, or `release/10` worktrees.
- You want a repeatable, low-risk flow that preserves local work and only proposes focused dependency updates.
- You need standardized per-repo reporting and PR creation when fixes are applied.

## Repositories
Each repository has a branch-based working directory structure.

Default `master` repositories use the `-master/` subfolder:
- `/Users/lli/GitWorkspace/case-map-editor/case-map-editor-master/`
- `/Users/lli/GitWorkspace/cms-editor/cms-editor-master/`
- `/Users/lli/GitWorkspace/database-editor/database-editor-master/`
- `/Users/lli/GitWorkspace/dataclass-editor/dataclass-editor-master/`
- `/Users/lli/GitWorkspace/form-editor/form-editor-master/`
- `/Users/lli/GitWorkspace/monaco-yaml-ivy/monaco-yaml-ivy-master/`
- `/Users/lli/GitWorkspace/persistence-editor/persistence-editor-master/`
- `/Users/lli/GitWorkspace/process-editor/process-editor-master/`
- `/Users/lli/GitWorkspace/primefaces-themes/primefaces-themes-master/`
- `/Users/lli/GitWorkspace/restclient-editor/restclient-editor-master/`
- `/Users/lli/GitWorkspace/role-editor/role-editor-master/`
- `/Users/lli/GitWorkspace/runtimelog-view/runtimelog-view-master/`
- `/Users/lli/GitWorkspace/swagger-ui-ivy/swagger-ui-ivy-master/`
- `/Users/lli/GitWorkspace/ui-components/ui-components-master/`
- `/Users/lli/GitWorkspace/user-editor/user-editor-master/`
- `/Users/lli/GitWorkspace/variable-editor/variable-editor-master/`
- `/Users/lli/GitWorkspace/vscode-designer/vscode-designer-master/`
- `/Users/lli/GitWorkspace/webservice-editor/webservice-editor-master/`
- `/Users/lli/GitWorkspace/neo/neo-master/`

Default `release/12` repositories currently available in local `-12/` worktrees are:
- `/Users/lli/GitWorkspace/dataclass-editor/dataclass-editor-12/`
- `/Users/lli/GitWorkspace/form-editor/form-editor-12/`
- `/Users/lli/GitWorkspace/monaco-yaml-ivy/monaco-yaml-ivy-12/`
- `/Users/lli/GitWorkspace/primefaces-themes/primefaces-themes-12/`
- `/Users/lli/GitWorkspace/process-editor/process-editor-12/`
- `/Users/lli/GitWorkspace/swagger-ui-ivy/swagger-ui-ivy-12/`
- `/Users/lli/GitWorkspace/ui-components/ui-components-12/`
- `/Users/lli/GitWorkspace/variable-editor/variable-editor-12/`
- `/Users/lli/GitWorkspace/neo/neo-12/`

Default `release/10` repositories currently available in local `-10/` worktrees are:
- `/Users/lli/GitWorkspace/primefaces-themes/primefaces-themes-10/`
- `/Users/lli/GitWorkspace/process-editor/process-editor-10/`
- `/Users/lli/GitWorkspace/swagger-ui-ivy/swagger-ui-ivy-10/`

If the user supplies additional repository paths and a target branch-specific working directory is missing, skip that repository and report it as `not found`.

## Preconditions
1. `git`, `pnpm`, and `gh` are available in the shell.
2. The user has network access for `git pull` and dependency resolution.
3. The user has network access for PR creation.
4. The user confirms how to handle dirty worktrees before any revert/discard action.

## Audit Count Rules
- Use repo-scoped commands for all `pnpm` operations: `pnpm -C <repo-dir> ...`.
- When reporting CVE counts from `pnpm audit --json`, sum the numeric values in `metadata.vulnerabilities`.
- Do not report `n/a` if the JSON output is available and the vulnerability counts can be derived from it.
- If post-fix advisories remain, capture both the count and the remaining advisory details (CVE or GitHub advisory ID plus affected package).

## Procedure
Run the following for each target repository's branch-specific working directory.

1. Enter the branch working directory (e.g., `<repo>-master/`).
   - Confirm this is a valid git repository with `git status`.
  - Determine the intended base branch from the working-directory context.
  - For `*-master/` folders, the intended branch is `master`.
  - For `*-12/` folders, the intended branch is `release/12`.
  - For `*-10/` folders, the intended branch is `release/10`.
   - Verify the current branch matches the intended base branch before continuing. Do not use some other currently checked out feature branch just because it is active.
   - If the current branch does not match the intended base branch and the worktree is clean, switch to the intended base branch first.
   - If the current branch does not match and switching branches is blocked by local changes, stop and ask the user how to handle them.
   - If the folder doesn't exist or isn't a git repo, skip that repository and report it.
2. Check for uncommitted changes before branch sync.
	- If the worktree is clean: continue.
	- If the worktree is dirty: stop and ask the user whether to keep, stash, or discard changes.
	- If the user approves discarding only known remediation side effects, revert only the specific approved files instead of resetting the whole worktree.
	- Never discard or revert local changes without explicit user approval.

3. Sync latest baseline.
   - Confirm again that the current branch is the intended base branch for the working-directory context.
   - Pull latest changes from remote with fast-forward only: `git pull --ff-only`.
4. Baseline install and vulnerability scan.
	- Run `pnpm -C <repo-dir> install`.
	- Run `pnpm -C <repo-dir> audit --json` and record current findings.

5. Apply automated remediation attempt.
  - Run `pnpm -C <repo-dir> audit --fix=update`.

6. Re-scan vulnerabilities.
	- Run `pnpm -C <repo-dir> audit --json` again.
	- Compare with baseline to determine whether CVEs were reduced or removed.

7. Inspect resulting file changes.
	- Keep only dependency-related files required for the fix (for example lockfiles and package manifests).
  - Revert any `package.json` changes introduced by `pnpm audit --fix=update` when they only remove `workspace:` versions, since those changes are not needed.
  - Revert any `pnpm-workspace.yaml` changes introduced by `pnpm audit --fix=update` that add fixed packages to `minimumReleaseAgeExclude`, since the default release age should be sufficient.
  - After any such reverts, rerun `pnpm -C <repo-dir> install` so the lockfile is updated to match the final manifest state.
	- Exclude unrelated edits from the change set.

8. Create branch and PR when there are meaningful updates.
   - If no relevant changes exist, do not open a branch/PR.
   - If relevant changes exist (within the branch-specific working directory):
     - Create a new branch named like `chore/pnpm-audit-fix-transient-cve-<branch-token>-<date>`.
      - Use `master`, `release-12`, or `release-10` as the `<branch-token>` so branch names stay unique across worktrees.
      - Commit with a clear message describing audit-driven dependency updates.
      - Push branch and open one draft PR per repository working directory:
        - `git push -u origin <branch-name>`
        - Detect the current branch from the folder context (for example `master` for `<repo>-master/`, `release/12` for `<repo>-12/`, or `release/10` for `<repo>-10/`)
        - `gh pr create --draft --base <current-branch> --head <branch-name> --title "chore: pnpm audit fix transient CVEs" --body "<summary>"`
      - After the PR is created successfully, switch back to the default branch for that working directory and delete the temporary local branch:
        - `git checkout <current-branch>`
        - `git branch -d <branch-name>`
      - Verify the PR body before finishing so the reported audit counts are concrete and not `n/a`.
      - In the PR description, include:
        - baseline audit status
        - final audit status
        - files changed
        - any remaining unresolved CVEs
      - Format unresolved items as an indented sub-list, for example:

```md
## Summary
- Baseline audit status: 13 CVEs
- Final audit status: 1 CVE
- Files changed: pnpm-lock.yaml
- Remaining unresolved CVEs:
  - CVE-2026-34601: @xmldom/xmldom
```

## Decision Points
- Branch folder missing or not a git repo: skip that repository and report it.
- Dirty worktree detected: require explicit user decision before any destructive action.
- Branch-specific working directory is present but not on the expected branch: switch only if the worktree is clean; otherwise stop and ask the user how to handle local changes.
- No dependency change after fix: report only, skip branch and PR.

## Completion Criteria
- Every target repository has:
  - pre-fix audit result
  - post-fix audit result
  - change status (`no-change` or `changes-ready`)
  - PR link when changes were published
- Any repositories blocked by dirty worktrees are clearly listed with the pending user decision.
- Any repositories without the requested branch-specific working directory are clearly listed as skipped.

## Output Format
Return a compact table with one row per repository and these columns:
- repository (with branch suffix, for example `ui-components-master`, `ui-components-12`, or `process-editor-10`)
- git status (valid repo / not found / error)
- dirty worktree handled (yes/no + action)
- pre-fix CVE count
- post-fix CVE count
- changes detected (yes/no)
- PR URL (or `n/a`)
- notes
