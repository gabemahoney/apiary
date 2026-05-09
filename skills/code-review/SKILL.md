---
name: code-review
description: Perform code review of changed files after task completion. Returns a simple list of improvement work items.
---

## Overview

This skill performs code review on files changed during a Task. It
returns a list of improvement work items for the caller to review.
The skill is a thin orchestrator: it loads the project's
engineering best practices from the docs `apiary.md` points at, then
applies that guidance to the changed code.

Be thorough but not pedantic — focus on substance over style.

## Parameters

You will receive some instructions on which set of work to review.
It might be a ticket id from the project's ticket backend, or a git worktree.

## Your Mission

Analyze changed code files and return a focused list of actionable
improvement work items.

- Understand the work context from the user input.
- Review all commits and changed files.
- Focus only on source code files. Ignore natural-language
  documentation and unit-test code.
- If no code files were changed, output "No code files to review"
  and exit.

### Step 0: Load project engineering best practices

1. Read `apiary.md` `## Documentation Locations`.
2. Find the `Reference` entry for `Engineering best practices`.
3. If a path is configured, read that document; it is the
   review checklist for this run. Apply each rule it states to the
   changed files.
4. If the entry is `omitted`, missing, or unreadable, warn the user
   in your output that no engineering best practices doc was
   configured and fall back to the minimal generic check in Step 2.
   Do **not** substitute a hardcoded full opinion set — that
   defeats the purpose of the indirection.

### Step 1: Run linters / formatters (if configured)

If `apiary.md` `## Build Commands` configures a lint or format
command for the affected repo, run it. Note any issues that should
be fixed.

If no lint command is configured, skip this step.

### Step 2: Review changed files

For each changed file, read it and apply the rules from the
engineering best practices doc loaded in Step 0.

If no best-practices doc was configured, fall back to a minimal
generic check only:

- Correctness — does the code do what it claims to do?
- Obvious anti-patterns — dead / commented-out code, secrets in
  source, unparameterized queries against untrusted input,
  swallowed exceptions, missing error handling on critical paths.

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
## Code Review Work Items

1. <file:line> — <one-line description of the issue and the fix>
2. ...
```

If no items: output the heading and "No issues found." If the
guidance doc was missing, prefix the list (or the no-issues line)
with a single warning line, e.g.:

```
Note: no `Engineering best practices` doc configured in apiary.md
`## Documentation Locations`. Reviewed against minimal generic
checks only.
```
