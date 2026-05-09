---
name: project-setup
description: One-time configuration of a project for the Apiary workflow
---

# Overview

Run this skill once per project to bootstrap Apiary: detect the available
ticket-backend, create the three stores Apiary expects (Features, Plans, Bugs),
configure their status values, and write an `apiary.md` manifest at the
project root.

Re-running the skill in an already-configured project is safe — it detects
existing state, surfaces drift to the user, and changes nothing without
explicit approval.

Apiary supports two known ticket backends — **bees**
(https://github.com/gabemahoney/bees) and **beads**
(https://github.com/steveyegge/beads). This skill speaks generically about
"stores", "tickets", "tiers", and "status values"; map those to whichever
backend's tools are available at runtime.

# Entry gate

Before doing anything else, verify a backend is reachable — either as a
configured MCP server or as a CLI on PATH. If neither is present, tell the
user how to install one and exit:

- bees: https://github.com/gabemahoney/bees
- beads: https://github.com/steveyegge/beads

If exactly one backend is available, use it. If both are available, ask the
user which to use with `AskUserQuestion`.

# Steps

## 1. Determine the project root

Ask the user for the absolute path of the project root (the directory that
will own `apiary.md`). Suggest the current working directory as the default
but require confirmation. Repo discovery and any automatic root inference
are out of scope for this skill — a later setup phase will add that.

## 2. Check for an existing apiary.md

If `<project_root>/apiary.md` already exists, read it and treat this run
as an idempotent re-configuration:

- Parse the existing `## Project Root`, `## New Feature`, and `## New Bug`
  sections.
- Compare the parsed values to whatever the user provides in step 4.
- For any drift, use `AskUserQuestion` to confirm before overwriting.
- If nothing differs, report "already configured" and exit cleanly.

Treat the level-2 (`##`) headings as section markers. Preserve any unknown
sections you encounter — future skills will append more sections, and this
skill must not clobber them.

## 3. Discover existing stores

Query the backend for stores named `Features`, `Plans`, and `Bugs` (case
sensitivity follows the backend's own rules). For each:

- If missing, you will create it in step 5.
- If present, fetch its tier configuration and status values, and compare
  to the target configuration in step 5. Record any drift.

## 4. Interview the user

Use `AskUserQuestion` for choice prompts. Collect the minimum needed to
populate `apiary.md`:

- **Project root** — confirm or correct the path from step 1.
- **New Feature → source_references** — one of:
  - `github resolver` — the backend will resolve GitHub URLs/IDs itself.
  - `none, interview user` — the skill that creates a feature will ask
    the user for source material instead.
- **New Bug → source_references** — same two choices as above.

Resolve each value to a concrete string. Do not write `ask user` or any
other placeholder into `apiary.md` — the answer is captured here, in this
interview.

This skill intentionally does NOT ask about repo discovery, build
commands, project templates, documentation locations, or decision-tree
logic for source references. Those belong to later setup phases and will
be added by future skills as additional sections in `apiary.md`.

## 5. Create or reconcile the three stores

Target configuration:

| Store    | Child tiers                                     | Status values                                  |
|----------|-------------------------------------------------|------------------------------------------------|
| Features | t1: Doc / Docs                                  | `draft`, `ready`, `active`, `done`, `published` |
| Plans    | t1: Epic / Epics; t2: Task / Tasks; t3: Subtask / Subtasks | `draft`, `ready`, `active`, `done`             |
| Bugs    | (none)                                          | `open`, `active`, `done`, `published`          |

For each store:

- If it does not exist, create it with the listed tiers and configure its
  status values.
- If it exists with the listed configuration, leave it alone.
- If it exists with different tiers or statuses, present the diff via
  `AskUserQuestion` and only change what the user approves.

Use whichever backend tools and CLI are available; the skill stays
backend-agnostic so the LLM running it picks the right calls.

## 6. Write apiary.md

Write `<project_root>/apiary.md` with exactly these three top-level
sections, in this order:

```
# Apiary Configuration

## Project Root
<absolute path>

## New Feature
- source_references: <resolved value from interview>

## New Bug
- source_references: <resolved value from interview>
```

The `## Project Root` line is the only absolute path in the file. Any
future paths added by later skills must be relative to that root.

If you are updating an existing `apiary.md`, edit only the sections above
and preserve any other `##` sections verbatim — they belong to skills
that haven't been written yet.

# What comes next

Once setup is complete, point the user at `/new-feature` to start the
first feature in their project.
