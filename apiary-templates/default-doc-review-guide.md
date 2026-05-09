# Default Doc Review Guide

This is a default checklist used by the Apiary `doc-review` skill.
It is intentionally opinionated. Edit it or swap it for your own
document; whatever `apiary.md` `## Documentation Locations` →
`Doc review guide` points at is what `doc-review` will apply.

The companion `Doc writing guide` describes what good docs *look
like*; this guide describes what to flag during review.

## What to flag

### README (user-focused, concise)

- Install instructions out of date or wrong
- Setup / dependency steps no longer match the code
- Listed commands / sub-commands no longer match the code
- New top-level commands or APIs missing from the README
- Drift from the writing guide — implementation details, testing
  details, security details, or how-to-use-common-tools content
  that should not be in a README

### Architecture / contributor doc (cheat sheet for LLMs and new contributors)

- High-level architecture has changed and the doc has not
- New components have been introduced (or old ones removed) and the
  doc still describes the previous shape
- Schema or API endpoints have changed and are not reflected
- Data flow described is no longer accurate
- Code has crept into the doc (functions, methods, signatures);
  remove it
- "Why we did it this way" / trade-off rationale has crept in;
  move it to a design doc or remove it

### Inconsistencies anywhere

- Outdated commands still shown
- Deprecated features still documented as supported
- Changed file paths not updated
- Old config / schema formats shown as current

### Duplication and waste

- Sections that repeat information already present elsewhere —
  consolidate
- Verbose passages that can be compacted without losing meaning
- Docs serving the wrong purpose (e.g., README explaining
  architecture, architecture doc explaining how to install)

## Priority

1. **Critical:** new commands, changed APIs, altered user-visible
   workflows.
2. **Important:** component status, schema changes, new features
   not yet described.
3. **Nice-to-have:** clearer examples, tightened prose.

## What *not* to flag

- Minor formatting issues
- Grammar / wording nits
- Personal preferences

It is fine — and expected — for a review to return zero items. If
the docs accurately describe the current state, say so. Do not
invent gaps.

## Work-item quality

- Be specific: include file, section, and line number when possible.
- Be actionable: "Update `Quick Start` — add `new-cmd` usage" not
  "docs need work".
- Focus on user-visible and breaking changes.
