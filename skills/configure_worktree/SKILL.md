---
name: configure-worktree
description: Launch Claude in isolated tmux session to work on a Bee asynchronously
---

# Overview

Launch Claude Code in a dedicated tmux session within an isolated git worktree to work on a bee ticket autonomously.

# Usage

**Invocation patterns**:
- `/configure_worktree b.xxx`
- "Launch async session for b.yyy"
- "Make a workspace for b.xxx"

# Steps

## 1. Fetch and Display Bee Ticket

If the user did not provide a Bee to work on, check your context to see if there is a likely bee.
Suggest it to the user. If not, ask them what bee to configure the worktree to work on.
Use bees MCP tools to fetch the Bee ticket:

```bash
# Get ticket details
mcp_bees_show_ticket(ticket_id)
```

Display the ticket content to the user:
- Title
- Description  
- Labels
- Status
- Children (if any)
- Dependencies (if any)


## 2. Check current environment and ask user to confirm.

Check if a worktree already exists for this work. 
The worktree will be at the same directory level as the repo the user is in, named as the normalized ticket ID (dots replaced with underscores, e.g., `b.xxx` → `features_bees_abc`).
If it does, tell the user you will spawn the agent into that repo.
If it does not, ask the User for permission to setup that worktree

Use AskUserQuestion to confirm:

```
Setup work tree for {ticket_id}?

This will:
- Create a git worktree named {normalized_ticket_id} at ../ 

Proceed?
```

Options: Yes / No

If No, abort.

## 2a. If needed, Prepare Main Repo

Before creating worktree, ensure the current main repo is clean and committed:

```bash
git status
```

If there are uncommitted changes:
- Show what's changed
- Use AskUserQuestion: "Main repo has uncommitted changes. Should I commit them first?"
  - Options: "Yes, commit them" / "No, abort spawn"
- If yes, commit with descriptive message
- The worktree will be created from current HEAD

## 2b. If needed, create Worktree

Normalize ticket ID: Replace dots with underscores (e.g., `b.xxx` → `features_bees_abc`)

Determine the repo root from the current working directory:
```bash
git rev-parse --show-toplevel
```

Create the worktree as a sibling directory to the repo root:
```bash
git worktree add {repo_root}/../{normalized_ticket_id}
```

## 2c. Copy auto_approve.sh into worktree

`docker/` and `.claude/` are tracked in git and will be present in the worktree automatically. Only `auto_approve.sh` needs to be copied since it is gitignored:

```bash
MAIN_REPO="{repo_root}/../bees_main"
WORKTREE="{repo_root}/../{normalized_ticket_id}"

if [ -f "$MAIN_REPO/auto_approve.sh" ]; then
    cp "$MAIN_REPO/auto_approve.sh" "$WORKTREE/auto_approve.sh"
fi
```

If `bees_main` does not exist, try the repo root itself as the source:
```bash
cp "{repo_root}/auto_approve.sh" "{repo_root}/../{normalized_ticket_id}/auto_approve.sh"
```

## 3. Ask user for the agent's prompt

Use AskUserQuestion to get the full instruction string to pass to the spawned Claude agent. 
This can be a skill invocation, a freeform task description, or any valid prompt.

## 4. Ask model preference

Ask the User what model to use for the Claude Session: Opus or Sonnet

## 5. Launch via waggle

Use `mcp__waggle__spawn_agent` to launch Claude in the worktree. This handles tmux session creation and environment setup correctly.

```
mcp__waggle__spawn_agent(
    repo="{repo_root}/../{normalized_ticket_id}",
    session_name="{normalized_ticket_id}",
    agent="claude",
    model="{model}",
    command="{user_prompt}"
)
```

## 6. Get session status with waggle

Use the waggle MCP server to report back the status of the session you just spawned in the next step.

## 8. Report Status

Provide summary:

```
✅ Async session spawned successfully

Session name: {normalized_ticket_id}
Worktree: {repo_root}/../{normalized_ticket_id}
Ticket: {ticket_id}
Prompt: {user_prompt}

To attach manually: tmux attach-session -t "{normalized_ticket_id}"
```

