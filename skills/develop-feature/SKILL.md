---
name: develop-feature
description: Multi-Epic orchestrator that walks a Feature's Plan, dispatches /develop-epic per Epic, and surfaces combined review feedback at the end
---

## Overview

`develop-feature` is the user-facing entry point for executing a
fully-decomposed Plan. It is a thin coordinator over `develop-epic`:
it gates the Feature, locates the Plan, drives Plan and Feature
through the generic `ready -> active -> done` flow, and dispatches
`/develop-epic` per child Epic in dependency order. Between Epics it
asks the user whether to continue or pause. At close-out it presents
the combined deferred review feedback that `develop-epic`
accumulated across all Epics.

This skill writes Plan and Feature status only. Epic, Task, and
Subtask status are owned by `develop-epic` and the subagents it
dispatches.

## Status Ownership

This skill OWNS, and only OWNS, these transitions:

- Feature -> `active` (idempotent; on entry once the gate passes)
- Plan -> `active` (idempotent; on entry once the gate passes)
- Plan -> `done` (idempotent; at close-out, after every Epic is
  `done`)
- Feature -> `done` (only when the user explicitly opts in via the
  close-out prompt; this is user-driven, the skill executes the
  user's choice)

This skill does NOT touch:

- Epic status (owned by `develop-epic`)
- Task status (owned by `develop-epic`)
- Subtask status (owned by the dispatched subagents)
- PRD, SRD, or Bug status
- Any non-Feature ticket type at the top level

`draft` and `ready` are not written by this skill.

## Resume Invariant

All transitions are idempotent. State lives entirely in ticket
status — there is no separate progress file, no rollback step, and
no scratch state on disk.

- Re-invoking the skill on a Feature already in `active` picks up at
  the next non-`done` Epic in dependency order.
- Re-invoking on an Epic that is already `active` (because a
  previous run was interrupted mid-Epic) simply re-dispatches
  `/develop-epic` — that skill's own resume logic handles in-Epic
  recovery.
- `done` Epics are skipped silently on resume.
- If the user paused between Epics, Feature stays `active`, Plan
  stays `active`, and completed Epics stay `done`. No cleanup is
  required before resume.

## Argument

`argument-hint: "<feature-id>"`

Accept the Feature ID as the sole argument. If omitted, query the
ticket backend for Features with status `ready` or `active` and ask
the user which one to work on.

### Step 0: Read `apiary.md`

Read `<project_root>/apiary.md` first. Extract:

- **Project Root** — used as the base for any path operations and
  passed through to `/develop-epic`.
- **Documentation Locations** — captured for context; passed
  through to `/develop-epic` (which, in turn, passes them to its
  subagents).

If `apiary.md` is missing, emit a one-line notice ("apiary.md not
found; proceeding without project context") and continue. The
skill remains functional without it; downstream `/develop-epic`
invocations will run with the same caveat.

### Step 1: Gate the Feature

Look up the Feature by ID and inspect its status:

- `ready` or `active` — proceed (`active` is the resume case).
- `draft` — refuse with an actionable message: the Feature is not
  yet promoted. Point the user at `/req-review` for review-driven
  promotion or at manual promotion to `ready`. Do not modify the
  Feature.
- `done` or `published` — report cleanly that the Feature is
  already complete and exit. Do not modify any tickets.
- Any other state — refuse with a clear explanation of which
  statuses are accepted (`ready`, `active`).

### Step 2: Plan-by-Feature lookup (load-bearing)

Each Feature has exactly one Plan. `write-plan` records the
Feature's ticket ID on the Plan as a source reference, and this
skill queries that back-pointer to find the Plan from the Feature.

Query the ticket backend for Plans whose source reference value
equals the Feature's ticket ID. Use generic phrasing in any
user-facing output: "look up the Plan whose source reference is the
Feature's ticket ID". Do not name the storage field directly.

The concrete query mechanism is chosen by the LLM at runtime based
on which ticket-backend tools are available. The skill does not
prescribe a tool name; it prescribes the contract: filter the Plans
store by source-reference value equal to the Feature ticket ID.

Outcomes:

- **Zero matches** — error with a clear message that no Plan exists
  for this Feature, and point the user at `/write-plan`. Do not
  modify any tickets.
- **Exactly one match** — proceed with that Plan.
- **More than one match** — refuse defensively. Creating a second
  Plan for the same Feature is not supported by the upstream
  authoring skill, so multi-match indicates manual data corruption
  the user must resolve. Report the conflicting Plan IDs and stop.

### Step 3: Set Feature -> `active`, Plan -> `active`

Both transitions are idempotent — if the ticket already carries the
target status, do not re-write it.

- Set Feature -> `active`.
- Set Plan -> `active`.

Do not touch Epic, Task, or Subtask status here. Per Status
Ownership, those belong to `develop-epic` and the subagents it
dispatches.

### Step 4: Per-Epic loop (in dependency order)

1. Enumerate the Plan's Epic children.
2. Sort them topologically by `up_dependencies`. Never select an
   Epic whose up-dependencies are not all `done`.
3. Walk the sorted list:
   - If the Epic is `done`, skip silently (resume case) and
     continue to the next Epic.
   - If the Epic is `draft`, refuse: the Epic has not been written
     yet. Point the user at `/write-epic` and stop. This is a
     planning gap, not an execution gap, so the skill does not
     attempt to recover.
   - Otherwise (`ready` or `active`), invoke `/develop-epic
     <epic-id>`. Pass the Epic ID; do not pass any other
     orchestration state. `/develop-epic` reads `apiary.md` itself.
   - **Do not set Epic status here.** Per Status Ownership,
     `/develop-epic` owns Epic transitions. This skill never writes
     Epic, Task, or Subtask status under any code path.
   - **Capture the deferred-feedback accumulator** returned by
     `/develop-epic`. Record per-Epic:
     `{epic_id, epic_title, deferred_items: [...]}`. Append the
     record to a combined accumulator keyed by Epic ID.
   - **Do not modify, deduplicate, or re-triage the items.**
     `/develop-epic` has already filtered them through its
     review/fix loop and iteration cap; this skill is a passthrough
     for presentation.

4. After each Epic completes (and before invoking the next one),
   if any non-`done` Epics remain, ask the user:

   - Question: "Continue to next Epic `<id>` '`<title>`'?"
   - Options:
     - "Continue" — proceed to the next Epic.
     - "Pause" — exit cleanly. Feature stays `active`, Plan stays
       `active`, completed Epics stay `done`. The user can
       re-invoke `/develop-feature <feature-id>` later to resume
       at the next non-`done` Epic.

5. If `/develop-epic` was interrupted mid-Epic and the Epic is
   still `active` on the next run, simply re-invoke
   `/develop-epic <epic-id>` — that skill's own resume logic picks
   up in-Epic. This skill performs no in-Epic recovery itself.

### Step 5: Close-out (when every Epic in the Plan is `done`)

1. **Set Plan -> `done`** (idempotent).

2. **Present the aggregated deferred review feedback** to the user,
   organized by Epic. Use a clear heading per Epic
   (`<epic-id> — <title>`) and list each deferred item with its
   reviewer source and reason for deferral, exactly as
   `/develop-epic` returned them. State explicitly that **this is
   the only place this feedback is shown** — `/develop-epic`
   accumulates the items but deliberately does not present them to
   the user, so suppressing this step would lose the information.

3. **Ask about Feature -> `done`** — ask the user:

   - Question: "Mark Feature as done?"
   - Options:
     - "Yes, mark done" — set Feature -> `done`. Per Status
       Ownership, Feature -> `done` is normally user-driven; the
       user opting in via this prompt counts as user direction, so
       the skill executes their choice.
     - "No, leave as active" — leave the Feature `active` so the
       user can continue testing or follow up. The skill exits
       cleanly with no further status writes.

4. **Suggest the next step.** Print:

   ```
   Test your Feature, then run `/teardown_worktree` when ready.
   ```

   Do not invoke `/teardown_worktree` automatically — worktree
   teardown is owned by a separate skill and runs on the user's
   timeline.

## Out of Scope (explicit non-responsibilities)

- **Commits.** This skill never creates commits. The
  one-commit-per-Epic policy is owned by `/develop-epic`.
- **Format and full test.** `/develop-epic` runs Format and the
  Full test suite per Task and at Epic close-out. This skill never
  runs them.
- **Worktrees.** Worktree setup and teardown are owned by separate
  skills (`/configure_worktree`, `/teardown_worktree`). This skill
  never touches worktrees.
- **Epic, Task, and Subtask status.** Owned by `/develop-epic` and
  its subagents. This skill never writes them.
- **Non-Feature ticket types.** This skill operates only on
  Features and their Plans. It does not act on PRDs, SRDs, Bugs,
  or any other top-level ticket type.

## Constraints

- Generic backend language only. The skill speaks of Features,
  Plans, Epics, Tasks, Subtasks, source references, and tickets.
- Generic statuses only: `draft`, `ready`, `active`, `done`,
  `published`.
- All status transitions are idempotent.
- The skill does not prescribe specific ticket-backend tool names;
  it prescribes contracts (status reads, status writes, the
  Plan-by-Feature source-reference query) and the LLM picks the
  tools available at runtime.
- The skill never asks the user about deferred review items
  individually — they are presented as an aggregated block once at
  close-out.
- The skill never pushes to remote.
