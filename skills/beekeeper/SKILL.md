---
name: beekeeper
description: Orchestrate the SDLC across multiple repos by launching and monitoring Claude Code instances to do work
---

# Beekeeper

## Overview

Beekeeper is the top-level SDLC orchestrator. It surveys multiple repos for ready work,
launches Claude Code instances to execute that work autonomously, and monitors their progress.

It operates at the fleet level — across repos, across hives, across agents — so the user
can manage an entire development portfolio from a single session.

This skill depends on bees and waggle to be installed.

## Invocation Patterns

- `/beekeeper` — survey all managed repos and present the work queue


---

## Workflow

### 1. Discover Managed Repos
Run the bees list-hives command. Every hive returned is one you manage.
Do this every time /beekeeper is called. Do not use cached data from your context.
You may have context that limits your current scope to a subset of these. If so, only report and act on this subset.
If you do not have such context, report and act on the whole set.

### 2. Survey All Repos

If your context limits you to a subset of repos, skip the broad queries below and query only the specific hives for those repos.

Otherwise, run these five queries in parallel to sweep all repos at once.
Each query uses inline `report` fields so no follow-up `show_ticket` call is needed.

```yaml
# Query 1: All Idea bees
stages:
  - [type=bee, hive~ideas]
report: [title, ticket_status, up_dependencies]

# Query 2: Doc children of Ideas (PRDs, SRDs)
stages:
  - [type=bee, hive~ideas]
  - [children]
report: [title, ticket_status, parent]

# Query 3: All Plan bees
stages:
  - [type=bee, hive~plans]
report: [title, ticket_status, children, up_dependencies]

# Query 4: Epics under Plans (children of Plan bees)
stages:
  - [type=bee, hive~plans]
  - [children]
report: [title, ticket_status, parent]

# Query 5: All Bug bees
stages:
  - [type=bee, hive~bugs]
report: [title, ticket_status]
```

Do this every time /beekeeper is called. Do not use cached data from your context.

**Linking results across queries:**
- **PRD/SRD → Idea**: Match Query 2 results to Ideas via the `parent` field.
- **Plan → Idea**: Match Query 3 results to Ideas via the Plan's `up_dependencies` field.
- **Epic status → Plan**: Match Query 4 results to Plans via the `parent` field to determine 🟢/🔴/🐝 for the Plan column.

Group results by repo using the hive's `scope` field from `list_hives`.

### 3. Check Worktrees and Active Sessions

Run in parallel:

**A. Worktree check** — for each Plan Bee:
1. Normalize the Plan Bee ID (replace dots with underscores, e.g. `b.2e8` → `b_2e8`)
2. Derive the scope parent from the hive's scope field (strip `/**`, e.g. `/Users/gmahoney/projects/bees_project/**` → `/Users/gmahoney/projects/bees_project`)
3. Construct the candidate path: `{scope_parent}/{normalized_id}`
4. Check if it is a valid git worktree:
```bash
git -C {candidate_path} rev-parse --git-dir 2>/dev/null
```
If the command succeeds, the worktree exists. If it fails, it does not.

**B. Active sessions** — call `mcp__waggle__list_agents` once.
For each Plan Bee with a worktree, check if any agent's `repo` field matches the worktree path (`{scope_parent}/{normalized_id}`).

### 4. Status

Group status by Repo. Show first a table for Ideas then one for Bugs:

For Ideas, show a table with these columns:
ID | Title | Status | SRD | PRD | Plan | Worktree

Status emojis:
Idea: ☠️ for finished, 🟢 for pupa
SRD: 🟢 for pupa or ❌ for missing
PRD: 🟢 for pupa or ❌ for missing
Plan: 🐝 for worker, 🟢 if all Epics are pupa, 🔴 if any Epics are larva, ❌ for no Plan
Worktree: 🐝 if active waggle session in worktree, 💤 if worktree exists but no session, ❌ if no worktree, — if no Plan

- Report the status of each Bug
☠️ for finished, 🟢 for Open
