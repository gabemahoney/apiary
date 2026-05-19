---
name: develop-epic
description: Execute one Epic end-to-end via subagents, with PM oversight, review cycle, and a single commit
---

## Overview

`develop-epic` is the per-Epic execution engine. It runs all Tasks in
one Epic in dependency order, dispatches subagents (engineer,
test-writer, doc-writer, plus a per-Task pm) to do the work, runs
a post-Tasks review cycle (code-reviewer, test-reviewer,
doc-reviewer), and lands a **single commit per Epic** at close-out.
It accumulates deferred reviewer feedback for the caller (typically
`develop-feature`) — it never surfaces deferred items to the user.

This skill drives the Epic and Task statuses through the generic
flow `ready` -> `active` -> `done`. All transitions are idempotent.
On resume, statuses alone are the state.

## Status Ownership

This skill OWNS, and only OWNS, these transitions:

- Epic -> `active` (idempotent; on entry)
- Epic -> `done` (idempotent; at close-out, after final commit)
- Task -> `active` (idempotent; on Task entry)
- Task -> `done` (idempotent; after Task's Subtasks done and Format +
  Full test pass)
- Subtask -> `active` (immediately before dispatching each subagent)
- Subtask -> `done` (immediately after the subagent returns successfully)

This skill does NOT touch:

- Plan status (owned by Plan-level skills and the Feature-execution
  skill)
- Feature status (owned by Feature-level skills)
- PRD/SRD status
- Bug status

`draft` and `ready` are not written by this skill.

## Argument

`argument-hint: "<epic-id>"`

Accept the Epic ID as the sole argument. If omitted, ask the user
for it.

### Step 0: Read `apiary.md`

Read `<project_root>/apiary.md` first. It supplies:

- **Project Root** — used as the base for `git` operations and to
  locate Documentation Locations.
- **Documentation Locations** — `Reference` (engineering best
  practices, test guide, doc writing guide) and `Maintained`
  (contributor docs, customer-facing docs). These paths are passed
  through to dispatched subagents in their spawn prompts.
  Categories listed as `omitted` are skipped gracefully when
  referenced downstream.
- **`## Build Commands`** — per-repo build commands. Record the
  **Format** and **Full test** slots; they are used in step 4 (per
  Task) and step 6 (close-out). Categories listed as `omitted` are
  skipped with a one-line notice.

If `apiary.md` is missing, emit a one-line warning ("apiary.md not
found; proceeding without project context") and continue. Subagents
will still run but without Documentation Locations; Format and Full
test will be skipped wherever they would have fired.

### Step 1: Gate the Epic

Accept the Epic ID and look it up. Hard-fail with an actionable
message in any of these cases:

- Epic does not exist.
- Epic status is `draft` or any value other than `ready` or `active`.
  Message: "Epic must be `ready` (or `active` to resume); run
  `/write-epic` to break it down first."
- Epic has any `up_dependencies` whose status is not `done`.
  Message: list the blocking IDs and titles, and instruct the user
  to either complete them first or run `/develop-feature` to
  coordinate Epic ordering across the parent Feature.

If status is `active`, treat this invocation as a **resume** —
preserve `active` Tasks and existing Subtask statuses; do not redo
Step 2 (codebase sync was already performed when the Epic first
went `active`).

Verify the seven subagent types are available for dispatch:
`engineer`, `test-writer`, `doc-writer`, `pm`, `code-reviewer`,
`test-reviewer`, `doc-reviewer`. If any are missing, instruct the
user to install them per the apiary repo README (section *Install
the subagents*) so the runtime picks them up.

### Step 2: Pre-execution codebase sync (ONCE per Epic, NOT per Task)

This step runs **once at Epic entry** and is **skipped on resume**
(when the Epic is already `active`).

For each completed `up_dependencies` Epic:

1. Walk the git log/diff covering changes since that Epic landed
   (use `git log` and `git diff` against the merge base or recorded
   commit, scoped to the project root recorded in `apiary.md`).
2. Read this Epic's Task and Subtask descriptions.
3. Where descriptions reference outdated reality — renamed files,
   removed functions, changed APIs, restructured modules — update
   the Task or Subtask body in place to match the new reality. Use
   the project's ticket-update mechanism for these edits.

Print a short summary of which Tasks/Subtasks were refreshed and
what changed. If nothing was stale, say "codebase sync: no
description updates needed."

### Step 3: Mark Epic active (idempotent)

Set Epic -> `active`. No-op if it is already `active`.

### Step 4: Per-Task loop (in dependency order)

Load the Epic's Tasks. Sort them by `up_dependencies` so no Task
runs while its prerequisites are unfinished. Skip Tasks that are
already `done`. For Tasks already `active` (resume case), pick up
from the first non-`done` Subtask of that Task.

For each Task that is not `done`:

1. **Set Task -> `active`** (idempotent).

2. **Dispatch implementer subagents per Subtask.** Read the Task's
   Subtasks. For each Subtask that is not `done`, decide its role
   from the Subtask body and dispatch the matching subagent:

   - Implementation Subtasks -> dispatch the Engineer
   - Test Subtasks -> dispatch the Test Writer
   - Documentation Subtasks -> dispatch the Doc Writer

   Pass the Subtask body, Documentation Locations, and Build
   Commands from `apiary.md` into each spawn prompt. This skill
   writes Subtask `→ active` immediately before dispatching each
   subagent and Subtask `→ done` immediately after the subagent
   returns successfully.

   Track which subagent types ran during this Epic — the set is
   used to gate reviewer dispatch in step 5.

3. **Dispatch the per-Task `pm` subagent.** Spawn one `pm`. The
   `pm` reads:

   - The Task and all its Subtasks
   - The parent Epic
   - The parent Plan
   - The Feature's PRD and SRD via the Plan -> Feature
     source-reference chain (resolver per `apiary.md`)

   The `pm` advises on scope creep, alignment with the PRD/SRD,
   and design questions raised by the implementers. The `pm` is
   advisory only — `develop-epic` (this skill) makes the final
   triage decision on every item the `pm` raises.

4. **Run Format + Full test** from `apiary.md`'s `## Build
   Commands`. For multi-repo projects, run them per repo. If a
   slot is `omitted` or `apiary.md` was missing, emit a one-line
   notice ("skipping <Format|Full test>; not configured") and
   advance.

5. **On test pass:** set Task -> `done` (idempotent). **Do NOT
   commit at this stage.** Advance to the next Task.

6. **On test fail:** leave Task at `active`. Either respawn the
   appropriate fix subagent (engineer, test-writer, or doc-writer
   depending on the failure surface) and re-run Format + Full
   test, or stop and surface the error to the user. Do NOT
   advance to the next Task on failure.

**Resume contract.** If `develop-epic` is re-invoked on a
partially-executed Epic (status `active`), the loop walks Tasks
in order; the first Task that is not `done` is the entry point.
Inside that Task, the first Subtask that is not `done` is the
entry point. Status alone is the state — there is no separate
run-log file. Tasks already `done` are skipped without
re-dispatching subagents.

### Step 5: Post-Tasks review cycle (conditional dispatch)

Once every Task in the Epic is `done`, run the review cycle.
Reviewer dispatch is conditional on which writer types ran during
this Epic (tracked in step 4):

- Dispatch the Code Reviewer only if at least one Engineer
  subagent ran. The reviewer wraps the apiary `/code-review`
  skill.
- Dispatch the Test Reviewer only if at least one Test Writer
  subagent ran. The reviewer wraps `/test-review`.
- Dispatch the Doc Reviewer only if at least one Doc Writer
  subagent ran. The reviewer wraps `/doc-review`.

Run the dispatched reviewers concurrently.

**Triage.** Each reviewer returns a numbered list of work items.
For each item, `develop-epic` decides:

- **Act now** — respawn the corresponding implementer subagent
  (`engineer`, `test-writer`, or `doc-writer`) to fix the item,
  then re-run Format + Full test for the affected Task(s). On a
  non-trivial fix, re-run the relevant reviewer.
- **Defer** — keep the item but do not act on it. Add a record to
  the deferred-feedback accumulator (see below).

**Stalemate exit.** Continue the review/fix loop in a lane while the reviewer's feedback is substantively changing across iterations. When iteration N's feedback is substantively unchanged from iteration N-1's, the lane is stalemated. Add unresolved items in that lane to the deferred-feedback accumulator with an explicit `impasse: true` flag (alongside `reviewer`, `item_text`, `reason_deferred`) so `develop-feature` can render them distinctly.

**Deferred-feedback contract** (load-bearing for the caller):

- Accumulate `{reviewer, item_text, reason_deferred}` records for
  every deferred item.
- Write the accumulator to the Epic ticket body as a `## Deferred
  Review Feedback` block (rewriting the block on each update so
  resume keeps the freshest set). This is the resume-safety
  channel.
- Return the same accumulator to the caller (`develop-feature`,
  Epic 8) as the primary mechanism. Epic 8 owns user
  presentation.
- Do **NOT** ask the user about deferred items here. `develop-epic`
  never surfaces deferred review feedback to the user.

### Step 6: Close-out (single commit per Epic)

If the Epic is already `done` on entry to this step (re-invocation
on an Epic that previously closed), no-op with the message "Epic
already done" and exit cleanly.

Otherwise:

1. **Final validation.** Run Format then Full test from
   `apiary.md`'s `## Build Commands` (per repo for multi-repo).
   Skip with a one-line notice if `omitted` or `apiary.md` was
   missing.

2. **On Full test fail:** do **NOT** commit. Surface the failure
   to the user and stop. Epic stays `active`.

3. **On Full test pass — single commit per Epic:**

   - Stage every modified file under the project's git repo(s).
     Use `git add -A` (or the per-repo equivalent) so this captures
     skill changes, source code, tests, docs, and — if the ticket
     store lives inside this repo — the ticket store contents.
   - Create **ONE commit** covering the entire Epic. The commit
     message names the Epic ID, the Epic title, and lists the
     completed Task IDs.
   - **NEVER** push to remote. This skill commits only.

4. **Set Epic -> `done`** (idempotent). Do NOT touch Plan or
   Feature status under any code path.

5. **Return value.** Return to the caller:

   - The deferred-feedback accumulator from step 5.
   - A one-paragraph Epic summary (Tasks completed, files
     changed, deferred review items count, design questions the
     `pm` raised).

6. **Exit suggestions.** Print:

   ```
   Epic done. Run `/develop-feature <feature-id>` to advance to the
   next Epic in the Feature, or run `/develop-epic <next-epic-id>`
   directly, or stop here.
   ```

