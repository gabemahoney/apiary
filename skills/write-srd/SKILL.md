---
name: write-srd
description: Write a Software Requirements Document (SRD) from a Product Requirements Document (PRD). Adds or modifies an SRD doc as a Doc child of a Feature ticket in the Features store, alongside the PRD. Explores the codebase, then defines what must be true about the software without solving the problem.
---

# Write SRD from PRD

## Overview

Given a PRD, write a companion Software Requirements Document (SRD) that defines
what must be true about the software for the PRD to be satisfied. The SRD does not
solve the problem — it provides guidance on constraints the implementation must meet.

The SRD is a Doc child of the same Feature ticket the PRD is a child of (the PRD
is a sibling, not the parent).

Be definitive in the SRD. Do not present options unless you believe further research is needed
- Flag such comments as **RESEARCH NEEDED**

## Status Ownership

This skill creates the SRD doc with status `draft` (idempotent — setting
`draft` when it is already `draft` is a no-op). It does NOT set the SRD to
`ready` — that transition is owned by `/req-review`. It does NOT set the
parent Feature ticket's status.

## Input

The user may invoke `/write-srd` with one of:

- A **Feature ticket ID** (most common) — find the PRD as a sibling Doc
  child of that Feature ticket.
- A **PRD ticket ID** — walk up to the parent Feature ticket, then locate
  the PRD (which is the ticket that was passed in).
- **Nothing** — ask the user for the Feature ticket whose PRD child should
  drive the SRD.

In every case, the SRD is added as a Doc child of the Feature ticket,
alongside the PRD.

## Workflow

### 0. Read `apiary.md`

Read `<project_root>/apiary.md` first. It supplies:

- **`## New Feature` section** — declares whether a `source_references`
  resolver is configured for Feature tickets (e.g., `github resolver`).
  This is what tells you how source references on the parent Feature
  ticket should be resolved when seeding the SRD draft.
- **Documentation Locations** — `Reference` (engineering best practices,
  test guide, doc writing guide). Categories listed as `omitted` are
  skipped gracefully when referenced downstream. Pull the "doc writing
  guide" location for use while drafting the SRD.

If `apiary.md` is missing, emit a one-line notice ("apiary.md not found;
proceeding without project context") and continue. Source-reference
pre-population in step 2 will be skipped, and any doc writing guidance
will fall back to defaults.

### 1. Determine the Feature ticket and locate the PRD

Resolve the Feature ticket and PRD according to the **Input** rules above.
View the Feature ticket to confirm it exists and to read its body. View
the PRD Doc child as well — it is the primary source for the SRD.

### 2. Pre-populate from source references

If the parent Feature ticket has a source reference recorded AND
`apiary.md`'s `## New Feature` section declares a resolver:

- Use the configured resolver to fetch the referenced content (for
  example, `gh issue view <id>` for `github resolver`). Seed the SRD
  draft with anything the source already states about constraints,
  affected components, data shapes, or non-functional requirements.
- If the source reference is absent, the resolver is not configured, or
  the fetch fails, emit a one-line notice ("source reference not
  resolved; proceeding with PRD and ticket body only") and continue
  using the PRD and Feature ticket body as the only seed context.

This step is gracefully optional and never blocks the SRD on an
unresolved source reference. It is **distinct from** "Read the PRD"
(step 3): the PRD is a sibling Doc, not the parent, so it is read
separately from the parent Feature's source reference.

### 3. Read the PRD

Read the PRD ticket thoroughly. Understand:
- What is being built
- What components are affected
- What edge cases exist
- What acceptance criteria are defined

### 4. Explore the Codebase

Before writing anything, explore the codebase to understand:
- **Architecture**: Top-level directory structure, module organization
- **Affected modules**: Which files/modules will need changes based on the PRD
- **Existing patterns**: How similar features are currently implemented (config loading, validation, error handling, serialization, etc.)
- **Data models**: Current model definitions that will be extended
- **Test fixtures**: How tests are structured, what helpers/factories exist, what constants are centralized
- **Configuration**: How config is loaded, validated, and accessed

Explore the codebase before drafting — read top-level structure, the modules the PRD will touch, and existing patterns for similar features. Be thorough; the SRD must reference real modules and conventions from the actual codebase.

#### Style Rules

- **No source code.** No code examples, no function signatures, no pseudocode.
- **No line numbers.** You can mention files but not lines, these are going to change during execution.
- **Define what must be true**, not how to implement it.
- **Be specific.** Reference actual module names and patterns from the codebase.
- **Be concise.** Each requirement should be one clear statement.
- **Be testable.** Every requirement should be verifiable.

#### Structure

Organize requirements by domain/concern, not by PRD section. Use numbered requirement
groups with subsections:

```
## SR-1: [Domain Name]

### SR-1.1: [Specific Concern]
- Requirement statement
- Requirement statement

### SR-1.2: [Another Concern]
- Requirement statement
```

#### Required Sections

Derive these from the PRD. Common patterns include:

- **Data Model** — Fields, types, constraints, serialization, which entities are affected
- **Configuration** — Schema changes, validation rules, resolution/fallback logic
- **API / Commands** — New endpoints or tools, parameters, return types, registration
- **Execution / Processing** — Runtime behavior, lifecycle, external process management
- **Validation / Linting** — New rules, integration with existing validation, corruption marking
- **Existing Behavior** — What must NOT change, what must be preserved
- **Test Fixtures** — How to extend existing test infrastructure (see below)
- **Documentation** — What docs must be written after implementation

#### Test Fixtures Section (Required)

Every SRD must include a test fixtures section. This prevents implementors from
hardcoding values and breaking test conventions. The section must:

1. **Identify the test architecture** — Where fixtures, helpers, constants, and docs live
2. **Mandate use of existing helpers** — Name the specific factory functions and fixtures
   that must be used instead of hardcoding
3. **Specify helper extensions** — What parameters need to be added to existing helpers
   for the new feature
4. **Specify new fixtures** — What new fixtures are needed and where they should be defined
5. **Specify test organization** — Which existing test files new tests belong in vs.
   new test files needed

The goal: an implementor who reads only the SRD test section should know exactly which
helpers to call, which fixtures to use, and where to put new tests — without hardcoding
any YAML, JSON, or string literals in test files.

### 5. Write the SRD

Add the SRD as a Doc child of the Feature ticket (sibling to the PRD).
Use the title "SRD" by default; note that downstream consumers identify SRD content by what's in the body, not by exact-matching this title.
Set its status to `draft` (idempotent — if an SRD already exists at `draft`, this is a no-op). Do NOT set the SRD to `ready`; that is `/req-review`'s job. Do NOT change the parent Feature ticket's status.

### 6. Present to User

After writing the SRD, present a summary of the requirement groups to the user.
List each SR group with a one-line description.

## What NOT to Include

- Source code or pseudocode
- Implementation details (which functions to call, which libraries to use)
- Step-by-step implementation plans
- Time estimates
- Architectural proposals or design alternatives

## Quality Checklist

Before presenting the SRD, verify:

- [ ] Every PRD acceptance criterion maps to at least one SR
- [ ] Every SR is testable and specific
- [ ] No contradictions between SRD and PRD
- [ ] Codebase references are accurate (real module names, real fixture names)
- [ ] Test fixtures section names specific helpers, factories, and constants files
- [ ] No source code anywhere in the document
- [ ] Requirements define "what must be true" not "how to implement"

### 7. Next steps

Ask the user:
- Question: "SRD is saved at status `draft`. What next?"
- Options:
  - "req-review" — review the SRD (and PRD) for completeness and promote
    it to `ready`
  - "mark as ready" — escape hatch. The user takes responsibility for
    promoting the Feature to `ready` themselves (typically by manually
    marking it via the ticket backend). **This skill does NOT set any
    status to `ready`** — SRD `draft → ready` and Feature `draft → ready`
    are both owned by `/req-review`. Selecting this option skips both
    the in-skill next-step and `/req-review`; the user assumes
    responsibility for the gate `/write-plan` enforces.
