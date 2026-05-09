---
name: write-prd
description: Write a Product Requirements Document (PRD) for a development effort. Adds or modifies a PRD doc as a t1 Doc child of a Feature ticket in the Features store so downstream skills (write-srd, req-review, write-plan) can consume it.
---

# Write PRD

## Overview

The caller wants to write a PRD — a plain-english description of work to be
done, stated as requirements. The PRD is the origin document for a development
effort. Downstream skills will consume it:

- `/write-srd` — turns the PRD into software requirements
- `/req-review` — reviews the PRD for completeness and executability
- `/write-plan` — creates a Plan in the Plans store with Epics; auto-chains into `/write-epic` to add Tasks and Subtasks

The PRD must be thorough enough that an agent swarm can execute the work
autonomously, without needing to ask the user clarifying questions.

## Status Ownership

This skill creates the PRD doc with status `draft` (idempotent — setting
`draft` when it is already `draft` is a no-op). It does NOT set the PRD to
`ready` — that transition is owned by `/req-review`. It does NOT set the
parent Feature ticket's status.

## Workflow

### 0. Read `apiary.md`

Read `<project_root>/apiary.md` first. It supplies:

- **`## New Feature` section** — declares whether a `source_references`
  resolver is configured for Feature tickets (e.g., `github resolver`).
  This is what tells you how source references on the parent Feature
  ticket should be resolved.
- **Documentation Locations** — `Reference` (engineering best practices,
  test guide, doc writing guide). Categories listed as `omitted` are
  skipped gracefully when referenced downstream. Pull the "doc writing
  guide" location for use while drafting the PRD.

If `apiary.md` is missing, emit a one-line notice ("apiary.md not found;
proceeding without project context") and continue. Source-reference
pre-population in step 2 will be skipped, and any doc writing guidance
will fall back to defaults.

### 1. Determine the Feature ticket

The PRD will be a t1 Doc child of a Feature ticket in the Features store.
If the user did not provide a Feature ticket id, ask which Feature ticket
the PRD should be written under. View the ticket to confirm it exists and
to read its body before drafting.

### 2. Pre-populate from source references

If the parent Feature ticket has a source reference recorded AND
`apiary.md`'s `## New Feature` section declares a resolver:

- Use the configured resolver to fetch the referenced content (for
  example, `gh issue view <id>` for `github resolver`). Seed the PRD
  draft with that content as initial context for the body, problem
  statement, and acceptance criteria — anything the source already
  states.
- If the source reference is absent, the resolver is not configured, or
  the fetch fails, emit a one-line notice ("source reference not
  resolved; proceeding with ticket body only") and continue with the
  Feature ticket body as the only seed context.

This step is gracefully optional — never block the PRD on an unresolved
source reference.

### 3. Gather Requirements

You and the user may already have been chatting about a feature, and the
source reference (if any) may have seeded the draft. Fill out the PRD body
with as much information as you already have.

Then, interview the user to understand the work. Ask focused questions to
fill gaps so you can update the PRD.

Key questions to explore:
- What is the Problem Statement?
- What are the Acceptance Criteria?
- What is explicitly out of scope?
- Are there edge cases or error scenarios to handle?
- Are there UI/UX considerations?
- Are there performance or security constraints?
- Does this depend on or affect other work?

Use `AskUserQuestion` to gather information efficiently. Present options
where possible rather than open-ended questions.

#### Style Rules

- **Plain english.** No code, no pseudocode, no technical implementation details.
- **Concise and direct.** Less is more. Every sentence should add information.
- **Requirements, not solutions.** Describe what must be true, not how to build it.
- **Specific and testable.** Every requirement should be verifiable.
- **No user-story format required.** Just clear statements of what is needed.
- **Do not number product requirements.** Use plain conversational English to describe the problem.

#### Required Sections

```markdown
# [Feature/Project Name]

## Problem Statement
What problem exists today? Who has it? Why does it matter?
Keep this to 2-3 sentences max.

## Goals
What must be true when this work is complete?
Bulleted list of concrete outcomes.

## Non-Goals / Out of Scope
What this effort explicitly will NOT do.
Prevents scope creep during implementation.

## Requirements

### Functional Requirements
What the system must do. Each requirement should be:
- Specific enough to test
- Independent where possible

### Edge Cases and Error Handling
How the system should behave in non-happy-path scenarios.
Each edge case should describe the scenario and the expected behavior.

### Non-Functional Requirements (if applicable)
Performance, security, scalability, compatibility constraints.
Only include if relevant — do not pad with boilerplate.

### UI/UX Requirements (if applicable)
Layout, interaction, and presentation requirements.
Only include if the feature has a user-facing component.

## Acceptance Criteria
The checklist that must pass for this work to be considered done.
Each criterion should be observable and testable.
Format as a checklist:
- [ ] Criterion 1
- [ ] Criterion 2

## Assumptions
What is assumed to be true going into this work.
Dependencies on other systems, existing functionality, or prior work.

## Open Questions (if any)
Unresolved decisions flagged for user review.
Remove this section if there are none.
```

### 4. Present and Iterate

After writing the PRD, present a summary to the user:
- One-line description of each section
- Call out any assumptions made
- Highlight open questions

If the user has changes, iterate until they approve.

## Quality Checklist

Before presenting the PRD, verify:

- [ ] Problem statement is clear and concise
- [ ] Goals are concrete and measurable
- [ ] Non-goals section exists and prevents scope creep
- [ ] Every functional requirement is testable
- [ ] Edge cases and error scenarios are covered
- [ ] Acceptance criteria are specific and observable
- [ ] Assumptions are explicitly stated
- [ ] No implementation details or code anywhere in the document
- [ ] No contradictions between sections
- [ ] An agent could implement this without making assumptions

## What NOT to Include

- Source code, pseudocode, or technical implementation details
- Architecture decisions or design proposals (that's the SRD's job)
- Time estimates or scheduling
- Specific file paths, function names, or line numbers
- "How to build it" — only "what must be true"

### 5. Write the PRD

Add the PRD as a t1 Doc child of the Feature ticket. Use the title "PRD".
Set its status to `draft` (idempotent — if a PRD already exists at
`draft`, this is a no-op). Do NOT set the PRD to `ready`; that is
`/req-review`'s job. Do NOT change the parent Feature ticket's status.

### 6. Next steps

Use `AskUserQuestion` with:
- Question: "PRD is saved at status `draft`. What next?"
- Options:
  - "write-srd" — proceed to drafting the software requirements document
  - "req-review" — review the PRD for completeness and promote it to `ready`
  - "mark as ready" — skip review and let `/req-review` handle the
    transition later

This skill itself never sets the PRD to `ready`. The "mark as ready"
option is a hint to the user about the next state in the flow; the
actual transition is owned by `/req-review`.
