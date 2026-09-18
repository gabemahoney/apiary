---
name: apiary-setup
description: Configure hives for Apiary workflow
---

## Overview

This skill should be run once to configure a repo for the Apiary workflow. It can also be run to fix hives that
have gotten misconfigured.

## Valid configuration

The repo must have the following hives available and configured with these child tiers and valid status values:

### Bugs Hive
Child tiers:
none

Status values:
- open — open bug
- finished — done

### Ideas Hive
Child tiers:
- t1 - Doc / Docs

Status values:
- larva — not fully documented, not ready to work
- pupa — fully documented, ready to work
- worker — in progress
- finished — done

#### Plans Hive (nested inside Ideas)
Located at `{ideas_hive_path}/Plans`.

Child tiers:
- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

Status values:
- larva — not fully documented, not ready to work
- pupa — fully documented, ready to work
- worker — in progress
- finished — done

## Instructions

### Prerequisites

Before configuring hives, verify the following are installed and configured.

#### 1. bees

Verify bees is available either as a CLI on PATH (`which bees`) or as a configured MCP server. If neither is present, direct the user to the bees repo for installation instructions: https://github.com/gabemahoney/bees

#### 2. Apiary Agent Definitions

The Apiary workflow spawns role players as subagents defined in `agents/*.md` files. This is a **verify-only** step — no install, no repair.

Detect the skill install location and check that all 18 agent definition files are present at the parallel path:
- `<repo>/.claude/agents/` when the Apiary skills live in `<repo>/.claude/skills/`
- `~/.claude/agents/` when they live in `~/.claude/skills/`

The 18 files: `apiary-engineer.md`, `apiary-test-writer.md`, `apiary-doc-writer.md`, `apiary-product-manager.md`, `apiary-engineer-bugfix.md`, `apiary-test-writer-bugfix.md`, `apiary-doc-writer-bugfix.md`, `apiary-engineer-planner.md`, `apiary-test-writer-planner.md`, `apiary-doc-writer-planner.md`, `apiary-product-manager-planner.md`, `apiary-code-reviewer.md`, `apiary-test-reviewer.md`, `apiary-doc-reviewer.md`, `apiary-prd-writer.md`, `apiary-srd-writer.md`, `apiary-req-reviewer.md`, `apiary-plan-writer.md`.

If all 18 are present, report success and move on. If any are missing, report the missing filenames and instruct the user to re-run README Install step 2 (the paired copy of `skills/` and `agents/`). Perform no file copies, no backups, and no clone-path prompts.

#### 3. tmux (required for worktree skills)

The `configure_worktree` and `teardown_worktree` skills launch and manage Claude Code sessions in tmux and hard-depend on it. Check if tmux is installed (`which tmux`). If not, direct the user to install it via their system package manager (e.g. `apt install tmux`, `brew install tmux`).

#### 4. Repository Directory Structure

Apiary expects the repo's main checkout to live one level below a parent directory, so that git worktrees can be created alongside it during development:

```
project-root/
├── repo/          # the main checkout (already exists — do NOT create this)
└── ...            # worktrees appear here later, created by Apiary agents as needed
```

**Do not create the main repo or any worktree directories during setup.** The only check here is that the main checkout is not itself the top-level directory (i.e., it has a parent directory with room for sibling worktrees). If it is, offer to reorganize it before proceeding.

---

### Hive Configuration

#### Working directory requirement

Bees validates that hive paths are within the git repo root of the current working directory.

**Rule:** Before running `colonize-hive`, your CWD must be a directory that is at or above the hive path. This is typically:
- The **repo** — if the hive lives inside the repo (e.g. `<repo>/tickets/bugs`)
- The **project parent** — if the hive lives alongside the repo (e.g. `<project-parent>/Bugs`)

After colonization, `cd` into the target repo for all other bees write operations (`set_status_values`, `create_ticket`, etc.) — these need CWD inside the repo for write-permission checks.

**Do not** run bees write operations from the Apiary checkout — it is a queen repo without write permission.

If the MCP write tool rejects the call, fall back to the `bees` CLI invoked from the correct directory.

#### Scope requirement

When calling `colonize_hive`, **always pass an explicit `scope` parameter** that is specific to the target project, for example `scope="/home/user/projects/myproject/**"`. The default scope (`/home/user/**` or wider) overlaps with other projects' hives and the bees server will reject the creation with `cross_scope_hive_conflict` if any other project has already registered a hive with the same normalized name (e.g. `bugs`, `plans`).

Pick the narrowest scope glob that still covers the entire project directory tree — typically the project's parent directory with a trailing `/**`.

#### Create or validate

Check for the existence of the above hives and validate their configs.
If any hives are missing:
- **You must use `AskUserQuestion` to ask the user where each missing hive should live.** Do not assume a default path, do not reuse paths from a previous run, and do not proceed without an explicit answer. Suggest sensible options (e.g. `<repo>/tickets/bugs` in-repo, or `<project-parent>/bugs` sibling-to-repo) but always let the user pick.
- Once the user chooses a path, colonize the hive, passing:
  - `name` — the hive name (e.g. `Bugs`, `Ideas`)
  - `path` — absolute path where the hive will live (as answered by the user)
  - `child_tiers` — as defined in the Valid Configuration section above
  - `scope` — explicit project-scoped glob (see above)
- After colonization, set the hive's status values using `set_status_values` (remember the CWD requirement above).

If a hive exists:
- Validate its child tiers and status values.
- If they differ from above, ask user if you may change them to the values listed above.

### Documentation Locations

After hives are configured, ask the user to define their project documentation locations in their CLAUDE.md.
Note that these are repo-specific documents.

Use AskUserQuestion to ask: "Would you like to define repo-specific documentation locations in a repo-specific CLAUDE.md now?"
- Tell the user: "The Apiary workflow will enforce your project standards. To do so, you must define the locations of those documents in your CLAUDE.md. You may skip this step if you have not defined such standards."
- Options: "Yes" / "Skip for now"

If yes, ask for the following (can use a single multi-part question or sequential questions):
- Engineering best practices guide path
- Internal architecture docs directory path
- Customer-facing docs path (e.g. README)
- Test writing guide path
- Test review guide path
- Doc writing guide path

Do NOT volunteer the following context unless the user asks what a location is for:
- **Engineering best practices**: Used by the Engineer agent in fix-bug, hatch-epic, and do-bee to follow project coding standards when writing or modifying source code.
- **Internal architecture docs**: Used by the Engineer to understand existing system design, and by the Doc Writer to update architecture documentation after code changes.
- **Customer-facing docs**: Used by the Doc Writer to update user-facing documentation when user-visible behavior changes.
- **Test writing guide**: Used by the Test Writer to follow project testing conventions when writing or modifying tests.
- **Test review guide**: Used by the Test Writer to self-review test quality before completing work.
- **Doc writing guide**: Used by the Doc Writer to follow project documentation style and format conventions.

Then write or update a `## Documentation Locations` section in the project's CLAUDE.md with the provided paths, using this format:

```markdown
## Documentation Locations

- **Engineering best practices**: <path>
- **Internal architecture docs**: <path>
- **Customer-facing docs**: <path>
- **Test writing guide**: <path>
- **Test review guide**: <path>
- **Doc writing guide**: <path>
```
