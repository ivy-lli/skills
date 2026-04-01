---
name: ivy-pnpm-fix-transient-cve
description: 'Check ivy pnpm repositories for open CVEs in transient dependencies, apply safe audit fixes, validate results, and open PRs only when real dependency changes are introduced.'
argument-hint: 'Optional: run on all default ivy repos or a provided subset of absolute repo paths'
---

# Ivy PNPM Transient CVE Fix

## When To Use
- You need to scan ivy pnpm repositories for vulnerable transient dependencies that Renovate may not auto-upgrade.
- You want a repeatable, low-risk flow that preserves local work and only proposes focused dependency updates.
- You need standardized per-repo reporting and PR creation when fixes are applied.

## Repositories
Each repository has a branch-based working directory structure. Default repositories use the `-master/` subfolder:
- `/Users/lli/GitWorkspace/case-map-editor/case-map-editor-master/`
- `/Users/lli/GitWorkspace/cms-editor/cms-editor-master/`
- `/Users/lli/GitWorkspace/database-editor/database-editor-master/`
- `/Users/lli/GitWorkspace/dataclass-editor/dataclass-editor-master/`
- `/Users/lli/GitWorkspace/form-editor/form-editor-master/`
- `/Users/lli/GitWorkspace/persistence-editor/persistence-editor-master/`
- `/Users/lli/GitWorkspace/process-editor/process-editor-master/`
- `/Users/lli/GitWorkspace/restclient-editor/restclient-editor-master/`
- `/Users/lli/GitWorkspace/role-editor/role-editor-master/`
- `/Users/lli/GitWorkspace/runtimelog-view/runtimelog-view-master/`
- `/Users/lli/GitWorkspace/ui-components/ui-components-master/`
- `/Users/lli/GitWorkspace/user-editor/user-editor-master/`
- `/Users/lli/GitWorkspace/variable-editor/variable-editor-master/`
- `/Users/lli/GitWorkspace/vscode-designer/vscode-designer-master/`
- `/Users/lli/GitWorkspace/neo/neo-master/`

## Preconditions
1. `git` and `pnpm` are available in the shell.
2. The user has network access for `git pull` and dependency resolution.
3. The user confirms how to handle dirty worktrees before any revert/discard action.

## Procedure
Run the following for each target repository's branch-specific working directory.

1. Enter the branch working directory (e.g., `<repo>-master/`).
   - Confirm this is a valid git repository with `git status`.
   - The current branch should already be set to the intended branch (typically `master` when in the `-master/` folder).
   - If the folder doesn't exist or isn't a git repo, skip that repository and report it.
2. Check for uncommitted changes before branch sync.
	- If the worktree is clean: continue.
	- If the worktree is dirty: stop and ask the user whether to keep, stash, or discard changes.
	- Never discard or revert local changes without explicit user approval.

3. Sync latest baseline.
   - Confirm the current branch with `git status`.
   - Pull latest changes from remote: `git pull`.
4. Baseline install and vulnerability scan.
	- Run `pnpm install`.
	- Run `pnpm audit` and record current findings.

5. Apply automated remediation attempt.
	- Run `pnpm audit --fix`.
	- Run `pnpm install`.

6. Revert workspace side effects and normalize lock state.
	- If `pnpm-workspace.yaml` changed due to remediation, revert only that file.
	- Run `pnpm install` again after the revert.

7. Re-scan vulnerabilities.
	- Run `pnpm audit` again.
	- Compare with baseline to determine whether CVEs were reduced or removed.

8. Inspect resulting file changes.
	- Keep only dependency-related files required for the fix (for example lockfiles and package manifests).
	- Exclude unrelated edits from the change set.

9. Create branch and PR when there are meaningful updates.
   - If no relevant changes exist, do not open a branch/PR.
   - If relevant changes exist (within the branch-specific working directory):
     - Create a new branch named like `chore/pnpm-audit-fix-transient-cve-<date>`.
     - Commit with a clear message describing audit-driven dependency updates.
     - Push branch and open one PR per repository working directory:
       - `git push -u origin <branch-name>`
       - Detect the current branch from the folder context (e.g., `master` for `<repo>-master/`)
       - `gh pr create --base <current-branch> --head <branch-name> --title "chore: pnpm audit fix transient CVEs" --body "<summary>"`
     - In the PR description, include:
       - baseline audit status
       - final audit status
       - files changed
       - any remaining unresolved CVEs

## Decision Points
- Branch folder missing or not a git repo: skip that repository and report it.
- Dirty worktree detected: require explicit user decision before any destructive action.
- `pnpm-workspace.yaml` changed: revert that file and re-install before final audit.
- No dependency change after fix: report only, skip branch and PR.

## Completion Criteria
- Every target repository has:
  - pre-fix audit result
  - post-fix audit result
  - change status (`no-change` or `changes-ready`)
  - PR link when changes were published
- Any repositories blocked by dirty worktrees are clearly listed with the pending user decision.

## Output Format
Return a compact table with one row per repository and these columns:
- repository (with -branch suffix, e.g., ui-components-master)
- git status (valid repo / not found / error)
- dirty worktree handled (yes/no + action)
- pre-fix CVE count
- post-fix CVE count
- changes detected (yes/no)
- PR URL (or `n/a`)
- notes
