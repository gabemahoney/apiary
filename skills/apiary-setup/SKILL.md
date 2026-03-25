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

#### 2. Claude Code Agent Teams

The Apiary workflow uses agent teams to parallelize work. Check whether `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is set to `"1"` in `~/.claude/settings.json`. If not configured, direct the user to: https://code.claude.com/docs/en/agent-teams

#### 3. tmux (optional)

tmux enables split-pane display so each agent teammate gets its own pane. Without it, teammates run in-process (still functional). Check if tmux is installed (`which tmux`). If not, note it is optional but recommended and direct the user to the Claude Code agent teams documentation for setup guidance.

---

### Egg Resolver

Before configuring hives, set up the bees egg resolver.  
This will ensure your development work can be traced back to the requirements documents that defined it.  
Apiary uses a custom resolver you must download from the bees github repo.

Use AskUserQuestion to ask: "Where would you like to save the bees egg resolver in your repo?" (e.g. `resolvers/bee_resolver.py`).

Then:
1. Download the resolver from: `https://raw.githubusercontent.com/gabemahoney/bees/main/resolvers/bee_resolver.py`
2. Save it to the user-specified path within the repo
3. Make it executable: `chmod +x <path>`

When colonizing hives, pass the absolute path to this file as the `egg_resolver` parameter.
If hives already exist, update their `egg_resolver` configuration to point to this file.

### Hive Configuration

Check for the existence of the above hives and validate their configs.
If any hives are missing:
- Ask the user if you can create them and if so where they should reside.
- Colonize the hive with no optional repo_root (unless the user instructs you to)
- Configure the hive with valid child tiers and status values
- Pass the egg resolver path as `egg_resolver`

If a hive exists:
- Validate its child tiers and status values
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
