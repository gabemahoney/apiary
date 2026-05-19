---
name: fix-bug
description: Fix a bug described in a Bug ticket
---

## Overview

The user will either call without arguments or with a Bug ID.

- If called without arguments, find all Bug tickets for this repo and present
  them, asking which one to fix.
- If called with a Bug ID, find it in the Bugs store. If it is not in the
  Bugs store, tell the user and ask whether to continue.

You will ultimately get the Bug ID you need to work on.

### Setup

There must be a store called "Bugs". If it does not exist, ask the user
where to create one. It must have no child tiers and the following valid
status values: `open`, `active`, `done`, `published`.

The Bug status flow `fix-bug` drives is `open` -> `active` -> `done`. All
transitions are idempotent — setting a status to its current value is a
no-op. `published` is set by the user manually after shipping; this skill
never sets it.

### 1. Read `apiary.md`

Read `<project_root>/apiary.md` first. It supplies:

- **Documentation Locations** — `Reference` (engineering best practices,
  test guide, doc writing guide) and `Maintained` (contributor docs,
  customer-facing docs). Categories listed as `omitted` are skipped
  gracefully when referenced downstream.
- **`## New Bug` section** — declares whether a `source_references`
  resolver is configured (e.g., `github resolver`).
- **`## Build Commands`** — per-repo build commands, including the
  **Full test** slot used for pre-close validation (step 6).

If `apiary.md` is missing, emit a one-line notice ("apiary.md not found;
proceeding without project context") and continue. Documentation
Locations passed to subagents will be empty; pre-close validation in
step 6 will be skipped.

### 2. Validate Bug

#### Check status and dependencies

View the ticket. Check:

- The Bug's status. If `open`, update it to `active` (idempotent — skip
  if already `active`). If `active`, treat this invocation as a
  **resume** — do not reset status, do not duplicate setup; continue
  from the current state with the steps below (re-resolve source
  references, re-read Documentation Locations, continue with team
  formation).
- `up_dependencies`: any blockers must be in a state which says they
  are completed.

If blocked:

- Output blocking IDs and titles.
- Exit with message: "Cannot start Bug $1. It is blocked by: [list]".

#### Resolve source references

If the Bug has a source reference recorded and `apiary.md`'s `## New
Bug` section declares a resolver:

- Use the configured resolver to fetch the referenced content (for
  example, `gh issue view <id>` for `github resolver`). Add the fetched
  content to the working context for downstream steps and subagents.
- If the reference is absent, the resolver is not configured, or the
  fetch fails, emit a one-line notice ("source reference not resolved;
  proceeding with ticket body only") and continue with the Bug body as
  the only context.

### 3. Form Teams to Fix Bug

Analyze the bug, the source code, the tests, and the docs. Understand the
likely scope of the fix.

- If the fix requires modifications to source code, spawn an Engineer.
- If the fix requires modifications to unit tests, spawn a Test Writer.
- Always spawn a Doc Writer so it can determine if any docs need
  updating based on the changes.

Determine the scope and form the appropriate team. Do not ask for
confirmation.

Delegate all of the work to the team members dispatched above. Do not
take it on yourself.

Dispatch subagents to fix the bug. Spawn `engineer` if source code needs to change, `test-writer` if tests need to change, `doc-writer` if docs need to change. Always spawn `doc-writer`. Pass the Bug body and resolved source-reference content.

### 4. Review Loop

Once the team is done, dispatch the appropriate reviewers based on which
writers ran in step 3:

- If the Engineer ran, dispatch `code-reviewer` (which wraps `/code-review`).
- If the Test Writer ran, dispatch `test-reviewer` (which wraps `/test-review`).
- If the Doc Writer ran, dispatch `doc-reviewer` (which wraps `/doc-review`).

Get the feedback and make a judgement call about whether the suggested
work must be done.

- If so, **reform the first team** to do the work.
  - Re-dispatch the relevant subagents to address the review feedback.
    Do not do the work yourself.
  - If the feedback was minor enough, you may choose to **NOT** spawn
    every reviewer counterpart on this iteration. Spawn the team
    members required to do the work you deem necessary from the
    reviewer feedback.
- If not, move on to Final Review but you MUST share the ignored
  feedback for review.

Loop until the reviewer's feedback in iteration N is substantively unchanged from iteration N-1 — at that point, the lane is stalemated. Stalemated items flow into the existing **Final Review** payload that is returned to the caller. Do not prompt the caller mid-flight; the caller decides what to do with impasse items when it receives the work package.

### 5. Testing the bug

Ensure there is at least one unit test that fails before the bug fix and
passes after — this guards against re-introducing this regression.

### 6. Pre-close validation

Before flipping the Bug toward `done`, run the **Full test** command from
`apiary.md`'s `## Build Commands` section.

- For multi-repo projects, run the full-test command per repo.
- If the full-test slot is `omitted` or the `## Build Commands` section
  is absent, emit a one-line notice ("skipping pre-close validation;
  full test command not configured") and proceed.
- If a full-test command fails, return to step 3 (team formation) to
  address the failure before continuing.

### 7. Confirm the fix and close

Once the bug is fixed and pre-close validation has passed:

1. Present a concise summary of the changes (files changed, what was
   done, key behaviors).
2. Ask the user "Is this bug fixed?"
   - **Yes** — create one git commit for the fix using system or
     project-defined git guidance, then update the Bug status to `done`
     (idempotent — skip if already `done`), emit the final summary
     below, and exit.
   - **No** — loop back to step 3 (team formation / review). The Bug
     stays at `active`.
   - **No answer / user leaves** — exit cleanly with the Bug still at
     `active`. Do **not** set `done`. The next invocation of this skill
     on the same Bug will resume from step 2.

Final summary template:

```markdown
## Bug done: [bug-title]

**Bug**: <bug-id>
**Files Changed**: [count] files ([list key filenames if < 5, otherwise just count])
**Reviews**: [Code review: X issues found/None needed | Docs review: Y issues found/None needed]
**Ignored Review Feedback**: [list items that were flagged but not addressed, or "None"]
```
