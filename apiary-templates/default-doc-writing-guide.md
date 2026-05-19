# Default Doc Writing Guide

This is a default guide for writing and reviewing project documentation in an
Apiary project. It is consumed by the `doc-writer` subagent and `doc-review`
skill. It covers both how to write good docs and what to flag during review.
Edit it or swap it for your project's own document; whatever
`apiary.md` `## Documentation Locations` → `Doc writing guide`
points at is what the doc-related skills will apply.

Two audiences, two documents. Keep them separate.

## README — for human users

The README's only job is helping a human user install, configure,
and run the project. Optimize for that.

- **Include:** what the project is in one or two sentences;
  install steps; minimum configuration; the most common usage
  example; pointers to deeper docs.
- **Do not include:**
  - Implementation details — readers do not need them, and they rot.
  - Testing or unit-testing details — that belongs in a contributor
    or testing doc.
  - Discussion of security implications or threat models — that
    belongs in a security doc.
  - How to use common tools (shells, package managers, terminal
    multiplexers, etc.) — link out instead.
- **Keep it short.** A long README is usually a sign that other
  docs are missing.

## Architecture / contributor doc — for LLM coding agents and new contributors

The architecture doc is a *cheat sheet*. An LLM coding agent (or a
new human contributor) should be able to read it and have enough
context to navigate the codebase without re-reading every file.

- **Include:**
  - The list of logical components and what each one does
  - How components interact and how data flows between them
  - Use of external resources — databases, queues, file storage,
    third-party APIs
  - Schemas and API endpoints (current state only)
- **Do not include:**
  - Code — no functions, no methods, no signatures. The LLM can
    read the source if it needs to.
  - Performance bragging or rationalization
  - Test coverage numbers or testing strategy — that belongs in a
    testing doc
  - Design decisions, trade-offs, or "alternatives considered" —
    that belongs in design / ADR docs, not the architecture cheat
    sheet
  - History — what *used to be* the case. Describe only the
    current state.
  - Design-pattern names as decoration ("uses the Visitor
    pattern") — describe what the code does, not what pattern it
    namedrops.

## General principles

- Concise beats verbose. Cut anything you can cut.
- One source of truth per fact. If two docs both describe the same
  thing, pick one and link from the other.
- Examples are worth more than prose. Prefer a small concrete
  example over a paragraph of abstract description.
- Update docs in the same change that updates code. A doc that
  describes the previous behavior is worse than no doc.

---

## Reviewing Docs

### What to flag

#### README (user-focused, concise)

- Install instructions out of date or wrong
- Setup / dependency steps no longer match the code
- Listed commands / sub-commands no longer match the code
- New top-level commands or APIs missing from the README
- Drift from the writing guide — implementation details, testing
  details, security details, or how-to-use-common-tools content
  that should not be in a README

#### Architecture / contributor doc (cheat sheet for LLMs and new contributors)

- High-level architecture has changed and the doc has not
- New components have been introduced (or old ones removed) and the
  doc still describes the previous shape
- Schema or API endpoints have changed and are not reflected
- Data flow described is no longer accurate
- Code has crept into the doc (functions, methods, signatures);
  remove it
- "Why we did it this way" / trade-off rationale has crept in;
  move it to a design doc or remove it

#### Inconsistencies anywhere

- Outdated commands still shown
- Deprecated features still documented as supported
- Changed file paths not updated
- Old config / schema formats shown as current

#### Duplication and waste

- Sections that repeat information already present elsewhere —
  consolidate
- Verbose passages that can be compacted without losing meaning
- Docs serving the wrong purpose (e.g., README explaining
  architecture, architecture doc explaining how to install)

### Priority

1. **Critical:** new commands, changed APIs, altered user-visible
   workflows.
2. **Important:** component status, schema changes, new features
   not yet described.
3. **Nice-to-have:** clearer examples, tightened prose.

### What *not* to flag

- Minor formatting issues
- Grammar / wording nits
- Personal preferences

It is fine — and expected — for a review to return zero items. If
the docs accurately describe the current state, say so. Do not
invent gaps.

### Work-item quality

- Be specific: include file, section, and line number when possible.
- Be actionable: "Update `Quick Start` — add `new-cmd` usage" not
  "docs need work".
- Focus on user-visible and breaking changes.
