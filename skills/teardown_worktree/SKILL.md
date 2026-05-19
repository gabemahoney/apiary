---
name: teardown-worktree
description: Merge and clean up the git worktree(s) created by configure_worktree for a Feature, after the Feature is `done`
---

# Overview

Merge a Feature's worktree branch(es) back into each repo's target branch, clean up the worktree directories, and stop the launched session(s).

This skill is multi-repo aware via `apiary.md` and HARD-GATES on the Feature ticket having status `done`. It does NOT promote a Feature to `done` — that responsibility lives with `/develop-feature` close-out or manual user action.

**Important:** This skill must run from the **main repo of the project**, not from inside a worktree. If you are currently inside a worktree's launched session, stopping that session would terminate the process mid-execution. See Step 1 for how to handle this.

# Usage

**Invocation patterns**:
- `/teardown-worktree <feature-id>`
- `/teardown-worktree <task-id>` (skill walks up to the Feature)
- "Tear down the worktree for `<feature-id>`"
- "Merge and clean up the workspace for `<feature-id>`"

# Steps

## 0. Read apiary.md

Locate `apiary.md` in the Project Root. If absent, abort:

> `apiary.md` not found. Run `/project-setup` first.

Parse the `## Build Commands` section. Each `### <relative-path>` heading names a repo participating in this project. Resolve each `<relative-path>` to an absolute path (relative to the Project Root). Keep this list — it drives the multi-repo iteration in later steps.

## 1. Detect Execution Context

Determine whether the current working tree is inside a worktree spawned by `configure_worktree`:

```bash
git rev-parse --show-toplevel
```

If the resulting path matches `{repo_root}/../{normalized_ticket_id}` for any repo discovered in Step 0:

- Inform the user: "This skill must run from the project's main repo — teardown would stop this session."
- Instruct them to:
  1. Attach to or open a Claude session in the main repo
  2. Run `/teardown-worktree <feature-id>` from there
- Abort.

Otherwise continue.

## 2. Resolve the Feature Ticket and HARD GATE on `done`

The skill is invoked with a ticket ID. It may be a Feature, Plan, Epic, or Task ID. The skill must resolve to the **Feature ticket** — the canonical reference for teardown.

Resolution rules (walk up the parent chain as needed):

- Task → its parent Epic → that Epic's parent Plan → that Plan's source-reference ticket (= the Feature ID).
- Epic → its parent Plan → that Plan's source-reference ticket (= the Feature ID).
- Plan → its source-reference ticket (= the Feature ID).
- Feature → use as-is.

If no ticket ID was provided, ask the user which Feature to tear down (offer the Features that have sibling worktrees on disk in any repo from Step 0).

Read the resolved Feature ticket's `status` field.

### HARD GATE: Feature must be `done`

If the Feature's status is anything other than `done`:

- Print:

  > This Feature is `<status>`. Teardown is gated on Feature == `done`. Promote the Feature to `done` first — typically by completing all Epics via `/develop-feature` and confirming, or by manually marking it `done` if you've validated it. Then re-run `/teardown-worktree`.

- Ask the user to confirm the abort with a single option: **"OK, I'll promote and re-run"**. Do NOT offer an "ignore gate and proceed anyway" option. There is no override.
- Abort.

**The skill MUST NOT update the Feature's status to `done` itself.** Status promotion is owned by `/develop-feature` close-out (or manual user action), never by this skill.

If the Feature status is `done`, continue.

## 3. Build the Multi-Repo Teardown Set

Normalize the Feature ID for filesystem use (e.g., replace dots with underscores: `f.X9Q` → `f_X9Q`). Refer to the result as `{normalized_ticket_id}`.

For each repo discovered in Step 0:

- Compute the candidate sibling worktree path: `{repo_root}/../{normalized_ticket_id}`.
- Check whether it exists on disk and is registered as a worktree of `{repo_root}` (`git -C {repo_root} worktree list`).
- If it exists, add `{repo_root}` to the **teardown set**.
- If it does not exist, skip this repo (no worktree to tear down here).

If the teardown set is empty, abort:

> No worktrees found for `<feature-id>`. Nothing to tear down.

Multi-repo note: each repo proceeds independently up to and through the merge. Per-repo failure semantics are described in Step 7.

## 4. Per-Repo: Check for Uncommitted Changes

For each repo in the teardown set, run:

```bash
git -C {worktree_path} status
```

Collect the per-repo summaries into one consolidated overview, then ask the user once how to handle uncommitted changes. Allow per-repo decisions if the situations differ.

Per-repo options:
- **Commit them** — commit inside that worktree with a descriptive message.
  Exclude `auto_approve.sh` (it is gitignored and copied by `configure_worktree`, not a source change):
  ```bash
  git -C {worktree_path} add -A -- ':!auto_approve.sh'
  git -C {worktree_path} commit -m "uncommitted changes from <feature-id>"
  ```
- **Discard them** — `git -C {worktree_path} reset --hard HEAD`
- **Abort** — stop the entire teardown.

## 5. Per-Repo: Determine Target Branch and Confirm Merge Plan

Each repo may have its own target branch (`main`, `master`, `develop`, …). Do NOT assume a single global target branch.

For each repo in the teardown set, determine its target branch (typically the main repo's currently checked-out branch):

```bash
git -C {repo_root} rev-parse --abbrev-ref HEAD
```

Build a per-repo merge plan and present it to the user:

```
Multi-repo merge plan for Feature <feature-id>:

  <repo-a-relative-path>: merge `{normalized_ticket_id}` into `<target-a>`
  <repo-b-relative-path>: merge `{normalized_ticket_id}` into `<target-b>`

Proceed?
```

Options: **Yes** / **No** / **No, let me adjust target branches**.

If the user adjusts, capture per-repo overrides and re-confirm.

## 6. Per-Repo: Merge the Worktree Branch

For each repo in the teardown set, from `{repo_root}`:

```bash
git -C {repo_root} merge {normalized_ticket_id} --no-ff -m "Merge worktree <feature-id> into <target-branch>"
```

### Conflict policy (multi-repo)

If a merge conflict occurs in **any** repo:

- Run `git -C {repo_root} merge --abort` for the conflicting repo.
- **Abort the entire teardown.** Do not proceed with session-stop, worktree removal, or branch deletion in any repo — this avoids leaving the project in a partial-merge state.
- Print a per-repo status report:

  ```
  Teardown aborted due to merge conflict.

    <repo>     | merged | conflicted | not-attempted
    ----------- | ------ | ---------- | -------------
    <repo-a>   |   ✓    |            |
    <repo-b>   |        |     ✓      |
    <repo-c>   |        |            |       ✓
  ```

- Tell the user: "Resolve conflicts in `<repo>` and re-run `/teardown-worktree <feature-id>`. Already-merged repos will be detected as no-op on the next run."

If all merges succeed, continue.

## 7. Per-Repo: Stop the Launched Session

For each repo in the teardown set, stop the session that `configure_worktree` launched for this Feature in that repo.

**Do not hardcode a specific stop mechanism.** Inspect the project context to determine what `configure_worktree` originally used to launch the session, and use the corresponding stop mechanism. Examples:

- If the session was launched via tmux, stop it with `tmux kill-session -t <session>` (where `<session>` matches the convention `configure_worktree` used — typically derived from `{normalized_ticket_id}` and possibly the repo name).
- If the session was launched via the waggle MCP, use the waggle MCP teardown tool.
- If launched via any other mechanism, use that mechanism's equivalent stop operation.

Pick based on convention: configure_worktree labels each session with the normalized ticket ID (and repo name in multi-repo setups). Try the available mechanism's lookup first (waggle session-list, `tmux has-session`, etc.) and fall back to the next mechanism if absent.

## 8. Per-Repo: Remove the Worktree and Delete the Branch

For each repo in the teardown set:

```bash
git -C {repo_root} worktree remove {worktree_path} --force
```

Ask the user to confirm branch deletion (one consolidated prompt covering all repos in the teardown set). Then for each repo:

```bash
git -C {repo_root} branch -d {normalized_ticket_id}
```

If a branch delete fails (unmerged), warn per-repo and ask:
- **Force delete it** (`git -C {repo_root} branch -D {normalized_ticket_id}`)
- **Leave the branch**

## 9. Final Report

Per-repo outcomes in a table-like layout:

```
Teardown complete.

Feature: `<feature-id>` (status: `done` — gate-checked, not modified)

  Repo                 | Target branch | Merge      | Worktree removed | Session stopped
  -------------------- | ------------- | ---------- | ---------------- | ---------------
  <repo-a-rel-path>    | <target-a>    | merged     | yes              | yes
  <repo-b-rel-path>    | <target-b>    | merged     | yes              | yes
  <repo-c-rel-path>    | —             | skipped    | n/a              | n/a

Feature `<feature-id>` was `done` and is now merged across <n> repos.
```

The skill does **not** modify any ticket's status. The "status: `done`" in the report reflects what was *gate-checked* on entry, not a state transition performed by this skill.
