---
name: git-sync-push
description: >
  Safe Git sync-and-push workflow: fetches latest main, rebases current branch onto main,
  handles conflicts safely, and pushes (with user confirmation). Use this skill whenever the
  user wants to sync their branch with main and push, says things like "push my code",
  "sync and push", "update branch and push", "submit my changes", "push to remote",
  "rebase on main and push", or any variation of syncing a feature branch with main before pushing.
  Also triggers on Chinese phrases like "推送代码", "同步并推送", "提交推送", "推到远程".
  Do NOT use for simple `git commit` (no push intent), or for PR creation (use the PR workflow instead).
allowed-tools: Bash
---

# Git Sync & Push

Sync the current feature branch with the default branch and push to remote — safely and defensively.

**Core principle**: never lose uncommitted work, never corrupt branch history, always ask the user before irreversible actions.

> **Note**: Shell variables do not persist between separate Bash tool calls. Either combine dependent commands into a single call, or re-derive values (like `BRANCH_NAME`, `DEFAULT_BRANCH`) as needed in each call.

## Git Safety Protocol

- NEVER update git config
- NEVER run destructive commands (`--force`, `reset --hard`, `checkout .`) without explicit user request
- NEVER skip hooks (`--no-verify`) unless user asks
- NEVER force push to `main` / `master`
- NEVER auto-resolve merge conflicts — the user must approve the resolution strategy
- NEVER commit sensitive files (`.env`, credentials, private keys)
- If an operation fails, fix the root cause and retry — do NOT bypass safety checks

## Workflow

### Step 0 — Preflight checks

Gather repo state (these are fast, run sequentially in one Bash call):

```bash
git rev-parse --is-inside-work-tree &&
git branch --show-current &&
git remote -v &&
git status --porcelain &&
git log --oneline -5
```

Detect the default branch dynamically — do not hardcode `main`:

```bash
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
if [ -z "$DEFAULT_BRANCH" ]; then
  for candidate in main master; do
    if git rev-parse --verify "origin/$candidate" >/dev/null 2>&1; then
      DEFAULT_BRANCH="$candidate"
      break
    fi
  done
fi
[ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH="main"
```

**Gate checks** before proceeding:

| Check | Action if failed |
|-------|-----------------|
| Not a git repo | Stop, inform the user |
| `git branch --show-current` is empty | Stop — HEAD is detached, cannot sync a feature branch |
| Current branch is `$DEFAULT_BRANCH` or `master` | Stop — this workflow is for feature branches only |
| No remote `origin` | Stop, ask user to configure remote |
| Shallow clone (`git rev-parse --is-shallow-repository` = true) | Warn user, suggest `git fetch --unshallow` before proceeding |

### Step 1 — Safety snapshot

Uncommitted changes are the most vulnerable. Protect them before anything else.

```bash
BRANCH_NAME=$(git branch --show-current)
HEAD_BEFORE=$(git rev-parse HEAD)

# Detect whether stash actually creates an entry
STASH_COUNT_BEFORE=$(git stash list | wc -l | tr -d ' ')
git stash push -m "git-sync-push-backup-$(date +%s)" --include-untracked
STASH_COUNT_AFTER=$(git stash list | wc -l | tr -d ' ')
HAS_STASH=$( [ "$STASH_COUNT_AFTER" -gt "$STASH_COUNT_BEFORE" ] && echo true || echo false )
```

Record the values of `BRANCH_NAME`, `HEAD_BEFORE`, and `HAS_STASH` in the conversation context for use in later steps.

### Step 2 — Fetch latest remote refs

Fetch both the default branch and the current branch (if it exists on remote) to ensure all refs are up-to-date:

```bash
git fetch origin "$DEFAULT_BRANCH" "$BRANCH_NAME" 2>/dev/null || git fetch origin "$DEFAULT_BRANCH"
```

If fetch fails entirely (network error, auth issue, remote not found):
- Show the exact error to the user
- Suggest: check network, verify SSH key / token, run `git remote -v`
- **Stop and ask the user** — do NOT continue with stale data

### Step 3 — Rebase onto default branch

Rebase keeps history linear and clean — the standard practice for feature branches.

```bash
git rebase "origin/$DEFAULT_BRANCH"
```

**No conflicts?** Proceed to Step 4.

**Conflicts detected:**

1. List conflicted files: `git diff --name-only --diff-filter=U`
2. Show conflict markers with surrounding context for each file
3. Present resolution options to the user:

> **Important**: During `git rebase`, the semantics of `--ours` and `--theirs` are **swapped** compared to `git merge`.
> In a rebase, `--ours` refers to the branch being rebased **onto** (i.e., `origin/main`), and `--theirs` refers to **your current branch's commits** being replayed.

| Option | Command | Effect |
|--------|---------|--------|
| Resolve manually | User edits files, then `git add` + `git rebase --continue` | User controls everything |
| Keep main's version | `git checkout --ours <file>` | Discards your local changes for this file |
| Keep current branch's version | `git checkout --theirs <file>` | Discards main's changes for this file |
| Abort rebase | `git rebase --abort` | Safe — returns to pre-rebase state |

4. **Ask the user** which strategy to use — never decide autonomously
5. After resolution: `git add <files>` → `git rebase --continue`
6. If new conflicts appear in subsequent commits, repeat the process

**Other rebase failures** (dirty worktree, detached HEAD, etc.):
- Show the error, suggest `git rebase --abort`, ask before acting

### Step 4 — Restore stashed changes

**Only run if `HAS_STASH=true`** (stash was created in Step 1). If the worktree was clean, skip entirely — do NOT pop, as it could restore an unrelated old stash.

Use the named stash ref to avoid popping the wrong entry:

```bash
STASH_REF=$(git stash list | grep "git-sync-push-backup" | head -1 | cut -d: -f1)
[ -n "$STASH_REF" ] && git stash pop "$STASH_REF"
```

If pop causes conflicts:
- Show the conflicted files and ask for resolution guidance
- The stash is still safely stored even if pop fails — nothing is lost
- If the user wants to abandon stash recovery, they can use `git stash show -p` to review contents later, or `git stash branch <name>` to recover into a new branch

### Step 5 — Confirm and push

**Always ask the user before pushing.** First, determine if force-push is needed:

```bash
BRANCH_NAME=$(git branch --show-current)

# Canonical check: is the remote branch tip an ancestor of local HEAD?
# If not, history was rewritten and force-push is needed.
if git rev-parse --verify "origin/$BRANCH_NAME" >/dev/null 2>&1; then
  if git merge-base --is-ancestor "origin/$BRANCH_NAME" HEAD 2>/dev/null; then
    NEEDS_FORCE=false
  else
    NEEDS_FORCE=true
  fi
else
  NEEDS_FORCE=false
fi
```

Present a clear summary:

```bash
# Re-derive DEFAULT_BRANCH (shell vars don't persist across Bash calls; same logic as Step 0)
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
if [ -z "$DEFAULT_BRANCH" ]; then
  for candidate in main master; do
    if git rev-parse --verify "origin/$candidate" >/dev/null 2>&1; then
      DEFAULT_BRANCH="$candidate"
      break
    fi
  done
fi
[ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH="main"

git log "origin/$DEFAULT_BRANCH"..HEAD --oneline
git diff --stat "origin/$DEFAULT_BRANCH"..HEAD
git status --porcelain
```

Display to the user:
- **Branch**: `<branch-name>`
- **Commits ahead of default branch**: count + one-line list
- **Files changed**: stat summary
- **Uncommitted changes**: yes/no
- **Force push needed**: yes/no (and explain why — rebase rewrote history)
- **Command**: the exact push command that will be executed

Then ask: **"确认推送？(y/n)"**

On user confirmation:

```bash
if [ "$NEEDS_FORCE" = "true" ]; then
  git push --force-with-lease origin "$BRANCH_NAME"
else
  git push -u origin "$BRANCH_NAME"
fi
```

**If push is rejected** even with the chosen strategy:

| Option | Command | When to use |
|--------|---------|-------------|
| Safe force push | `git push --force-with-lease origin <branch>` | Rebase rewrote history, you're the only branch author |
| Pull and retry | `git pull --rebase origin <branch>` then push | Others are collaborating on this branch |
| Abort | — | Investigate first |

**Ask the user** which option to use. Do NOT force-push without explicit approval.

### Step 6 — Cleanup and confirm

After a successful push, drop all backup stashes created by this workflow:

```bash
MAX_ITERATIONS=20; ITER=0
while [ "$ITER" -lt "$MAX_ITERATIONS" ]; do
  STASH_REF=$(git stash list | grep "git-sync-push-backup" | head -1 | cut -d: -f1)
  [ -z "$STASH_REF" ] && break
  git stash drop "$STASH_REF" || break
  ITER=$((ITER + 1))
done
```

Report to the user:
- Branch name and pushed commit range
- Remote URL for quick reference
- Confirmation that stash backup has been cleaned up (or that none existed)

## Error Recovery

If anything goes wrong at any phase:

```bash
# 1. Abort in-progress rebase if any
git rebase --abort 2>/dev/null

# 2. Verify worktree is clean after abort; warn if residual changes exist
WORKTREE_STATUS=$(git status --porcelain)
if [ -n "$WORKTREE_STATUS" ]; then
  echo "WARNING: worktree has residual changes after rebase abort:"
  echo "$WORKTREE_STATUS"
  echo "Review these before restoring stash to avoid conflicts."
fi

# 3. Restore stashed changes (only if this workflow created one)
STASH_REF=$(git stash list | grep "git-sync-push-backup" | head -1 | cut -d: -f1)
if [ -n "$STASH_REF" ]; then
  if ! git stash pop "$STASH_REF"; then
    echo "WARNING: stash pop failed due to conflicts."
    echo "Your stash is still saved. Review with: git stash show -p $STASH_REF"
    echo "Or recover to a new branch: git stash branch recovery-branch $STASH_REF"
  fi
fi

# 4. Verify HEAD integrity
git rev-parse HEAD  # compare with HEAD_BEFORE recorded in Step 1
```

- Report what happened and the current repo state to the user
- Never leave the repo in a half-rebased or detached-HEAD state
- The user's uncommitted work is sacred — always restore it before stopping
