---
name: teardown-worktree
description: Merge and clean up a git worktree created by configure_worktree
---

# Overview

Merge a worktree branch back into the main repo, clean up the worktree directory, and kill the associated tmux session.

**Important:** This skill must run from the **main repo**, not from inside the worktree. 
If you are currently inside the worktree's tmux session, killing it would terminate the process mid-execution. See Step 1 for how to handle this.

# Usage

**Invocation patterns**:
- `/teardown_worktree b.Yk9`
- "Tear down the worktree for b.Yk9"
- "Merge and clean up workspace for b.Yk9"

# Steps

## 0. Determine worktree

If worktree not passed, find available worktrees via git and AskUserQuestion which one to teardown.

## 1. Detect Execution Context

Determine whether Claude is currently running inside the target worktree:

```bash
git rev-parse --show-toplevel
```

Compare the output path against the normalized ticket ID pattern (`{normalized_ticket_id}`).

**If running inside the worktree:**
- Inform the user: "This skill must run from the main repo — teardown would kill this session."
- Instruct them to:
  1. Attach to or open a Claude session in the main repo
  2. Run `/teardown_worktree {ticket_id}` from there
- Abort.

**If running from the main repo:** Continue.

## 2. Identify the Worktree

If the user did not provide a ticket ID, ask them for one.

Normalize ticket ID: Replace dots with underscores (e.g., `b.Yk9` → `b_Yk9`)

Confirm the worktree exists:
```bash
git worktree list
```

Also confirm the tmux session exists:
```bash
tmux list-sessions | grep "{normalized_ticket_id}"
```

If neither exist, inform the user and abort.

## 3. Check for Uncommitted Changes in the Worktree

```bash
git -C {worktree_path} status
```

If there are uncommitted changes:
- Show what's changed
- Use AskUserQuestion: "The worktree has uncommitted changes. What should I do?"
  - Options: "Commit them" / "Discard them" / "Abort"
- If commit: commit with a descriptive message inside the worktree.
  **Important:** Exclude `auto_approve.sh` — it is gitignored and copied by configure_worktree, not a source change.
  ```bash
  git -C {worktree_path} add -A -- ':!auto_approve.sh'
  git -C {worktree_path} commit -m "uncommitted changes from {ticket_id}"
  ```
- If discard: `git -C {worktree_path} reset --hard HEAD`
- If abort: stop.

## 4. Confirm Merge Target

Determine the main repo's current branch:
```bash
git rev-parse --abbrev-ref HEAD
```

Use AskUserQuestion to confirm:
```
Merge worktree branch into `{main_branch}`?

Worktree: {worktree_path}
Target branch: {main_branch}

Proceed?
```

Options: Yes / No / No, let me specify a branch

If the user specifies a different branch, use that instead.

## 5. Merge the Worktree Branch

The worktree branch name matches the normalized ticket ID (git worktree add creates a branch automatically). Merge it into the target branch from the main repo:

```bash
git merge {normalized_ticket_id} --no-ff -m "Merge worktree {ticket_id} into {main_branch}"
```

If merge conflicts occur:
- Report the conflicting files
- Use AskUserQuestion: "Merge conflicts detected. What should I do?"
  - Options: "Abort the merge" / "I'll resolve manually"
- If abort: `git merge --abort`, stop and tell the user to resolve manually before re-running teardown.

## 6. Kill the tmux Session

Only after a successful merge (or if no branch was merged), kill the tmux session:

```bash
tmux kill-session -t "{normalized_ticket_id}"
```

## 7. Remove the Worktree

```bash
git worktree remove {worktree_path} --force
```

Confirm branch deletion with AskUserQuestion. 

Then delete the branch:
```bash
git branch -d {normalized_ticket_id}
```

If the branch delete fails (unmerged), warn the user and ask:
- Options: "Force delete it" / "Leave the branch"

```
✅ Worktree torn down successfully

Ticket:       {ticket_id}
Merged into:  {main_branch}
Worktree:     {worktree_path} (removed)
tmux session: {normalized_ticket_id} (killed)
Ticket status: finished
```
