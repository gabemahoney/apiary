---
name: test-review
description: Review test files for quality, coverage, and correctness after task completion. Returns a simple list of improvement work items.
---

## Overview

This skill reviews test files changed during some period of work.
It returns a list of improvement work items for the caller to
review. The skill is a thin orchestrator: it loads the project's
test review guide (and, if helpful, test writing guide) from the
docs `apiary.md` points at, then applies that guidance to the
changed tests.

Be thorough but not pedantic — focus on substance over style.

## Parameters

You will receive some instructions on which set of work to review.
It might be a ticket id from the project's ticket backend, or a git worktree.

## Your Mission

Analyze changed test files and return a focused list of actionable
improvement work items.

- Understand the work context from the user input.
- Review all commits and changed test files.
- Focus only on test files. Ignore source code files and natural
  language documentation.
- If no test files were changed, output "No test files to review"
  and exit.
- You may presume previous agents left the tests in a working
  state; you do not need to run them.

### Step 0: Load project test guide

1. Read `apiary.md` `## Documentation Locations`.
2. Find the `Reference` entry for `Test guide`.
3. If a `Test guide` path is configured, read that document; it is
   both the writing guide (what good tests look like) and the review
   checklist for this run. Apply each rule it states to the changed
   tests.
4. If it is not configured, warn the user in your output that no
   test guidance doc was configured and fall back to the minimal
   generic check in Step 2. Do **not** substitute a hardcoded full
   opinion set — that defeats the purpose of the indirection.

### Step 1: Load the full test suite

Find all test files in the project and read all of them — not just
the changed ones. You need the full picture to identify cross-file
duplication, redundancy, and parameterization opportunities.
Changed files are your focus, but you can only spot bloat if you
know what already exists elsewhere.

### Step 2: Review changed test files

With the full suite loaded, review the changed files and apply the
rules from the test review guide loaded in Step 0.

If no guidance doc was configured, fall back to a minimal generic
check only:

- Correctness — do the assertions actually verify the claimed
  behavior?
- Obvious anti-patterns — tests with no assertions, swallowed
  failures, real network / filesystem I/O without mocking,
  hardcoded sleeps, tests of code that no longer exists.

That fallback is intentionally narrow. Anything richer should come
from the project's configured guidance doc.

### Step 3: Prioritize and filter

Each work item should be:

1. Actionable — can become a standalone Task
2. Specific — includes `file:line` where applicable
3. Important — not trivial
4. Concise — one-line description
5. Applicable — within the requirements; do not aim beyond what is
   needed

It is expected that many runs will return no items. That is OK —
do not invent issues. Reporting filler items risks an infinite
review loop, which is bad.

### Step 4: Generate work-item list

Output a simple numbered list directly in your response, in the
shape:

```markdown
## Test Review Work Items

1. <file:line> — <one-line description of the issue and the fix>
2. ...
```

If no items: output the heading and "No issues found." If the
guidance doc was missing, prefix the list (or the no-issues line)
with a single warning line, e.g.:

```
Note: no `Test guide` doc configured in apiary.md
`## Documentation Locations`. Reviewed against minimal generic
checks only.
```
