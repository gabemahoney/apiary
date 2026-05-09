# Default Doc Writing Guide

This is a default guide for writing project documentation in an
Apiary project. It is consumed by the `doc-writer` subagent (and by
`doc-review` when no project-specific guide is configured). Edit
it or swap it for your project's own document; whatever
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
