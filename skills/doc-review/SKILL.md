---
name: doc-review
description: Review documentation completeness after task completes. Checks that docs reflect changed behavior. Returns structured list of documentation work items.
---

## Overview

This skill reviews project documentation for completeness and
accuracy after a Task completes. The skill is a thin orchestrator:
it loads the project's doc review guide (and, if helpful, doc
writing guide) from the docs `apiary.md` points at, then applies
that guidance to the docs in the project.

Concise is better than verbose. Value brevity.

## Parameters

You will receive some instructions on which set of work to review.
It might be a Bees ticket id, or a git worktree.

## Your Mission

Analyze what changed, compare against current docs, return a list
of specific documentation gaps as work items.

### Step 0: Load project doc review guide

1. Read `apiary.md` `## Documentation Locations`.
2. Find the `Reference` entries for `Doc review guide` and
   `Doc writing guide`.
3. If a `Doc review guide` path is configured, read that document;
   it is the review checklist for this run. Apply each rule it
   states to the project's docs.
4. If a `Doc writing guide` is also configured, read it for
   additional context (the writing guide describes what *good* docs
   look like — useful when judging gaps).
5. If neither is configured, warn the user in your output that no
   doc guidance docs were configured and fall back to the minimal
   generic check in Step 3. Do **not** substitute a hardcoded full
   opinion set — that defeats the purpose of the indirection.

### Step 1: Identify maintained docs

From `apiary.md` `## Documentation Locations` `Maintained
Documentation`, find the paths for `Customer-facing docs` and
`Contributor docs`. Skip categories marked `omitted`.

These are the documents that may need to be updated as a result of
the changed code.

### Step 2: Understand what changed

Review all commits and changed files to understand the scope of
work: new features, changed behavior, new commands or APIs, config
or schema changes.

### Step 3: Review the maintained docs

Read each maintained doc and apply the rules from the doc review
guide loaded in Step 0 to flag gaps and inconsistencies created by
the changes from Step 2.

If no doc review guide was configured, fall back to a minimal
generic check only:

- Do the maintained docs still describe the project's current
  behavior, or have the recent changes made any part of them
  inaccurate?
- Are any new user-visible commands, flags, or endpoints missing
  from the customer-facing docs?

That fallback is intentionally narrow. Anything richer should come
from the project's configured guidance doc.

### Step 4: Prioritize and filter

Each work item should be:

1. Actionable — can become a standalone Task
2. Specific — includes `file:line` or section name where possible
3. Important — not trivial
4. Concise — one-line description
5. Applicable — within the requirements; do not aim beyond what is
   needed

It is expected that many runs will return no items. That is OK —
do not invent issues. Reporting filler items risks an infinite
review loop, which is bad.

### Step 5: Generate work-item list

Output a simple numbered list directly in your response, in the
shape:

```markdown
## Documentation Review Work Items

1. <doc>:<section or line> — <one-line description of the gap and the fix>
2. ...
```

If no items: output the heading and "No documentation issues
found." If the guidance doc was missing, prefix the list (or the
no-issues line) with a single warning line, e.g.:

```
Note: no `Doc review guide` doc configured in apiary.md
`## Documentation Locations`. Reviewed against minimal generic
checks only.
```
